<script>
    import { storage } from "$lib/firebase";
    import {
        ref,
        listAll,
        getDownloadURL,
        getMetadata,
        updateMetadata,
        deleteObject,
    } from "firebase/storage";
    import { onMount } from "svelte";

    let activeTab = $state("photos"); // photos, drawings
    let images = $state([]);
    let loading = $state(true);

    // --- IndexedDB for Image Blobs ---
    const DB_NAME = "GalleryCacheDB";
    const STORE_NAME = "imageBlobs";

    async function openDB() {
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(DB_NAME, 1);
            request.onerror = () => reject(request.error);
            request.onsuccess = () => resolve(request.result);
            request.onupgradeneeded = (event) => {
                const db = event.target.result;
                if (!db.objectStoreNames.contains(STORE_NAME)) {
                    db.createObjectStore(STORE_NAME);
                }
            };
        });
    }

    async function getCachedBlob(key) {
        try {
            const db = await openDB();
            const tx = db.transaction(STORE_NAME, "readonly");
            const store = tx.objectStore(STORE_NAME);
            return new Promise((resolve) => {
                const req = store.get(key);
                req.onsuccess = () => resolve(req.result);
                req.onerror = () => resolve(null);
            });
        } catch {
            return null;
        }
    }

    async function setCachedBlob(key, blob) {
        try {
            const db = await openDB();
            const tx = db.transaction(STORE_NAME, "readwrite");
            const store = tx.objectStore(STORE_NAME);
            store.put(blob, key);
        } catch (e) {
            console.warn("Storage quota exceeded or IDB error", e);
        }
    }

    async function fetchAllImages(directoryRef) {
        const res = await listAll(directoryRef);

        // Fetch files in current directory
        const filePromises = res.items.map(async (itemRef) => {
            const [url, metadata] = await Promise.all([
                getDownloadURL(itemRef),
                getMetadata(itemRef),
            ]);
            return {
                url,
                name: itemRef.name,
                fullPath: itemRef.fullPath,
                timeCreated: new Date(metadata.timeCreated).getTime(),
                size: metadata.size,
                contentType: metadata.contentType,
                isHidden: metadata.customMetadata?.hidden === "true",
            };
        });

        let currentFiles = await Promise.all(filePromises);

        // Recursively fetch from subdirectories
        const folderPromises = res.prefixes.map((folderRef) =>
            fetchAllImages(folderRef),
        );
        const subfolderFiles = await Promise.all(folderPromises);

        return [...currentFiles, ...subfolderFiles.flat()];
    }

    $effect(() => {
        const loadGallery = async () => {
            try {
                loading = true;
                images = []; // 탭 전환 시 상단에서 즉시 초기화하여 이전 잔상 제거

                // 1. Load from LocalStorage Cache first (Instant UI)
                const cacheKey = `gallery_meta_${activeTab}`;
                const cachedMeta = localStorage.getItem(cacheKey);
                if (cachedMeta) {
                    const parsed = JSON.parse(cachedMeta);
                    // Create object URLs for images already in IDB
                    for (let img of parsed) {
                        const blob = await getCachedBlob(img.fullPath);
                        if (blob) {
                            img.displayUrl = URL.createObjectURL(blob);
                        } else {
                            img.displayUrl = img.url;
                        }
                    }
                    images = parsed;
                    loading = false;
                }

                // 2. Fetch Fresh Data from Firebase
                const rootPath =
                    activeTab === "photos" ? "chat_images" : "drawings";
                const rootRef = ref(storage, rootPath);
                const allFiles = await fetchAllImages(rootRef);

                // Sort and process
                const freshImages = allFiles
                    .filter((file) => file.contentType?.startsWith("image/"))
                    .sort((a, b) => b.timeCreated - a.timeCreated);

                // 3. Update Blobs in Background (Non-blocking UI update)
                images = freshImages; // 먼저 목록을 보여줌
                loading = false;

                for (let img of freshImages) {
                    const cachedBlob = await getCachedBlob(img.fullPath);
                    if (cachedBlob) {
                        img.displayUrl = URL.createObjectURL(cachedBlob);
                    } else {
                        // Not in cache, download and store
                        try {
                            // CORS 이슈 등으로 fetch가 실패할 수 있으므로 타임아웃/에러 처리 강화
                            const controller = new AbortController();
                            const timeoutId = setTimeout(
                                () => controller.abort(),
                                5000,
                            );

                            const response = await fetch(img.url, {
                                signal: controller.signal,
                            });
                            clearTimeout(timeoutId);

                            const blob = await response.blob();
                            await setCachedBlob(img.fullPath, blob);
                            img.displayUrl = URL.createObjectURL(blob);

                            // 개별 이미지 로드 시 점진적으로 UI 업데이트
                            images = [...images];
                        } catch (e) {
                            console.warn(
                                `Failed to cache image ${img.name}:`,
                                e,
                            );
                            img.displayUrl = img.url;
                        }
                    }
                }

                // 4. Update state and Metadata Cache
                images = freshImages;
                localStorage.setItem(
                    cacheKey,
                    JSON.stringify(
                        freshImages.map((img) => {
                            const { displayUrl, ...rest } = img; // Don't cache temporary object URLs
                            return rest;
                        }),
                    ),
                );
            } catch (error) {
                console.error("Gallery fetch failed:", error);
            } finally {
                loading = false;
            }
        };

        loadGallery();
    });

    function formatSize(bytes) {
        if (bytes === 0) return "0 B";
        const k = 1024;
        const sizes = ["B", "KB", "MB", "GB"];
        const i = Math.floor(Math.log(bytes) / Math.log(k));
        return parseFloat((bytes / Math.pow(k, i)).toFixed(1)) + " " + sizes[i];
    }

    async function toggleHide(image) {
        const imageRef = ref(storage, image.fullPath);
        const newHiddenState = !image.isHidden;

        try {
            await updateMetadata(imageRef, {
                customMetadata: {
                    hidden: newHiddenState ? "true" : "false",
                },
            });

            // Update local state reactively
            image.isHidden = newHiddenState;
            images = [...images];
        } catch (error) {
            console.error("Toggle hide failed:", error);
            alert("상태 변경 중 오류가 발생했습니다.");
        }
    }

    async function deleteImage(image) {
        if (
            !confirm(`정말 이 이미지를 영구 삭제하시겠습니까?\n(${image.name})`)
        )
            return;

        const imageRef = ref(storage, image.fullPath);
        try {
            await deleteObject(imageRef);

            // Remove from local state
            images = images.filter((img) => img.fullPath !== image.fullPath);

            // Remove from IndexedDB
            const db = await openDB();
            const tx = db.transaction(STORE_NAME, "readwrite");
            tx.objectStore(STORE_NAME).delete(image.fullPath);

            // Update LocalStorage cache
            const cacheKey = `gallery_meta_${activeTab}`;
            localStorage.setItem(
                cacheKey,
                JSON.stringify(
                    images.map((img) => {
                        const { displayUrl, ...rest } = img;
                        return rest;
                    }),
                ),
            );

            alert("삭제되었습니다.");
        } catch (error) {
            console.error("Delete failed:", error);
            alert("삭제 중 오류가 발생했습니다.");
        }
    }
</script>

<section class="admin-section">
    <header class="section-header">
        <div class="header-main">
            <h2>갤러리 관리</h2>
            <div class="tabs">
                <button
                    class:active={activeTab === "photos"}
                    onclick={() => (activeTab = "photos")}
                >
                    사진 (채팅)
                </button>
                <button
                    class:active={activeTab === "drawings"}
                    onclick={() => (activeTab = "drawings")}
                >
                    그림 (사용자)
                </button>
            </div>
            <span class="count-badge">{images.length} 개의 이미지</span>
        </div>
    </header>

    {#if loading}
        <div class="loading-state">
            <div class="spinner"></div>
            <p>이미지 목록을 불러오는 중...</p>
        </div>
    {:else if images.length === 0}
        <div class="empty-state">
            <p>
                보관된 {activeTab === "photos" ? "사진이" : "그림이"} 없습니다.
            </p>
        </div>
    {:else}
        <div class="gallery-grid">
            {#each images as image (image.fullPath)}
                <div class="gallery-item" class:is-hidden={image.isHidden}>
                    <div class="image-wrapper">
                        <img
                            src={image.displayUrl || image.url}
                            alt={image.name}
                            loading="lazy"
                        />
                        <div class="image-overlay">
                            <div class="overlay-actions">
                                <a
                                    href={image.url}
                                    target="_blank"
                                    rel="noopener noreferrer"
                                    class="view-btn">원본 보기</a
                                >
                                <button
                                    class="hide-toggle-btn {image.isHidden
                                        ? 'shown'
                                        : 'hidden'}"
                                    onclick={() => toggleHide(image)}
                                >
                                    {image.isHidden
                                        ? "숨김 해제"
                                        : "갤러리 숨김"}
                                </button>
                                <button
                                    class="delete-btn"
                                    onclick={() => deleteImage(image)}
                                >
                                    영구 삭제
                                </button>
                            </div>
                        </div>
                        {#if image.isHidden}
                            <div class="hidden-badge">숨김됨</div>
                        {/if}
                    </div>
                    <div class="item-info">
                        <p class="filename" title={image.name}>{image.name}</p>
                        <div class="meta">
                            <span
                                >{new Date(
                                    image.timeCreated,
                                ).toLocaleDateString()}</span
                            >
                            <span>{formatSize(image.size)}</span>
                        </div>
                    </div>
                </div>
            {/each}
        </div>
    {/if}
</section>

<style>
    .admin-section {
        max-width: 1400px;
        margin: 0 auto;
    }

    .section-header {
        margin-bottom: 2rem;
    }

    .header-main {
        display: flex;
        align-items: center;
        gap: 2rem;
    }

    .section-header h2 {
        font-size: 1.75rem;
        color: #333;
        margin: 0;
    }

    .tabs {
        display: flex;
        background: #e9ecef;
        padding: 0.25rem;
        border-radius: 8px;
        gap: 0.25rem;
    }

    .tabs button {
        padding: 0.5rem 1.25rem;
        border: none;
        background: none;
        border-radius: 6px;
        font-size: 0.9rem;
        font-weight: 600;
        color: #666;
        cursor: pointer;
        transition: all 0.2s;
    }

    .tabs button:hover {
        color: #333;
    }

    .tabs button.active {
        background: #fff;
        color: #0066cc;
        box-shadow: 0 2px 4px rgba(0, 0, 0, 0.05);
    }

    .count-badge {
        background: #fff;
        padding: 0.4rem 1rem;
        border-radius: 99px;
        font-size: 0.85rem;
        font-weight: 700;
        color: #0066cc;
        border: 1px solid #e9ecef;
        margin-left: auto;
    }

    /* Loading State */
    .loading-state {
        text-align: center;
        padding: 5rem 0;
        color: #666;
    }

    .spinner {
        width: 40px;
        height: 40px;
        border: 4px solid #f3f3f3;
        border-top: 4px solid #0066cc;
        border-radius: 50%;
        margin: 0 auto 1.5rem;
        animation: spin 1s linear infinite;
    }

    @keyframes spin {
        0% {
            transform: rotate(0deg);
        }
        100% {
            transform: rotate(360deg);
        }
    }

    /* Gallery Grid */
    .gallery-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
        gap: 1.5rem;
    }

    .gallery-item {
        background: #fff;
        border-radius: 12px;
        overflow: hidden;
        box-shadow: 0 4px 15px rgba(0, 0, 0, 0.05);
        transition: transform 0.2s;
    }

    .gallery-item:hover {
        transform: translateY(-5px);
    }

    .image-wrapper {
        position: relative;
        aspect-ratio: 1 / 1;
        overflow: hidden;
        background: #f8f9fa;
    }

    .image-wrapper img {
        width: 100%;
        height: 100%;
        object-fit: cover;
        transition: transform 0.3s;
    }

    .image-overlay {
        position: absolute;
        top: 0;
        left: 0;
        width: 100%;
        height: 100%;
        background: rgba(0, 0, 0, 0.6);
        display: flex;
        justify-content: center;
        align-items: center;
        opacity: 0;
        transition: opacity 0.2s;
    }

    .gallery-item:hover .image-overlay {
        opacity: 1;
    }

    .overlay-actions {
        display: flex;
        flex-direction: column;
        gap: 0.75rem;
        padding: 1rem;
        width: 100%;
        align-items: center;
    }

    .view-btn,
    .hide-toggle-btn,
    .delete-btn {
        width: 80%;
        padding: 0.6rem;
        border-radius: 6px;
        font-size: 0.85rem;
        font-weight: 600;
        text-decoration: none;
        text-align: center;
        border: none;
        cursor: pointer;
        transition: all 0.2s;
    }

    .view-btn {
        background: #fff;
        color: #333;
    }

    .hide-toggle-btn.hidden {
        background: #ff4d4d;
        color: #fff;
    }

    .hide-toggle-btn.shown {
        background: #28a745;
        color: #fff;
    }

    .delete-btn {
        background: #212529;
        color: #fff;
    }

    .delete-btn:hover {
        background: #000;
    }

    .hidden-badge {
        position: absolute;
        top: 0.5rem;
        left: 0.5rem;
        background: rgba(255, 77, 77, 0.9);
        color: #fff;
        padding: 0.2rem 0.6rem;
        border-radius: 4px;
        font-size: 0.7rem;
        font-weight: 700;
        z-index: 2;
    }

    .gallery-item.is-hidden img {
        filter: grayscale(0.8) opacity(0.5);
    }

    .item-info {
        padding: 1rem;
    }

    .filename {
        font-size: 0.85rem;
        font-weight: 600;
        color: #333;
        margin: 0 0 0.5rem 0;
        white-space: nowrap;
        overflow: hidden;
        text-overflow: ellipsis;
    }

    .meta {
        display: flex;
        justify-content: space-between;
        font-size: 0.75rem;
        color: #888;
    }

    .empty-state {
        text-align: center;
        padding: 10rem 0;
        color: #999;
        background: #fff;
        border-radius: 16px;
    }
</style>
