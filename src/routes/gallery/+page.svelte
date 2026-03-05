<script>
    import { storage } from "$lib/firebase";
    import {
        ref,
        listAll,
        getDownloadURL,
        getMetadata,
    } from "firebase/storage";
    import { onMount } from "svelte";
    import { fade } from "svelte/transition";
    import { page } from "$app/state";

    let activeTab = $state("photos"); // photos, drawings
    let images = $state([]);
    let displayedImages = $state([]);
    let loading = $state(true);
    let expandedId = $state(null);
    let pageNum = $state(1);
    const pageSize = 10;
    let selectedImage = $state(null);

    // --- IndexedDB Cache (Open once at module level) ---
    const DB_NAME = "GalleryCacheDB";
    const STORE_NAME = "imageBlobs";
    let dbInstance = null;

    async function openDB() {
        if (dbInstance) return dbInstance;
        return new Promise((resolve, reject) => {
            const request = indexedDB.open(DB_NAME, 1);
            request.onerror = () => reject(request.error);
            request.onsuccess = () => {
                dbInstance = request.result;
                resolve(dbInstance);
            };
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
            console.warn("Storage quota exceeded", e);
        }
    }

    async function fetchAllImages(directoryRef) {
        const res = await listAll(directoryRef);
        const filePromises = res.items.map(async (itemRef) => {
            const [url, metadata] = await Promise.all([
                getDownloadURL(itemRef),
                getMetadata(itemRef),
            ]);
            // Only return NOT hidden images for user gallery
            if (metadata.customMetadata?.hidden === "true") return null;

            return {
                url,
                name: itemRef.name,
                fullPath: itemRef.fullPath,
                timeCreated: new Date(metadata.timeCreated).getTime(),
                contentType: metadata.contentType,
            };
        });

        let currentFiles = (await Promise.all(filePromises)).filter(
            (f) => f !== null,
        );
        const folderPromises = res.prefixes.map((folderRef) =>
            fetchAllImages(folderRef),
        );
        const subfolderFiles = await Promise.all(folderPromises);
        return [...currentFiles, ...subfolderFiles.flat()];
    }

    async function loadMore() {
        // loading 상태 중복 체크 대신 개별적 제어
        if (displayedImages.length >= images.length) return;

        const nextBatch = images.slice(
            displayedImages.length,
            displayedImages.length + pageSize,
        );

        // UI에 바로 빈 껍데기라도 추가하여 반응성 확보
        const startIdx = displayedImages.length;
        displayedImages = [...displayedImages, ...nextBatch];

        for (let i = 0; i < nextBatch.length; i++) {
            const img = displayedImages[startIdx + i];
            const cachedBlob = await getCachedBlob(img.fullPath);
            if (cachedBlob) {
                img.displayUrl = URL.createObjectURL(cachedBlob);
            } else {
                try {
                    const response = await fetch(img.url);
                    const blob = await response.blob();
                    await setCachedBlob(img.fullPath, blob);
                    img.displayUrl = URL.createObjectURL(blob);
                } catch (e) {
                    img.displayUrl = img.url;
                }
            }
            displayedImages = [...displayedImages]; // 개별 이미지 로드 시마다 업데이트
        }
    }

    $effect(() => {
        const initGallery = async () => {
            try {
                loading = true;
                images = [];
                displayedImages = [];

                // 1. LocalStorage에서 메타데이터 먼저 불러와 즉시 표시
                const cacheKey = `user_gallery_meta_${activeTab}`;
                const cachedMeta = localStorage.getItem(cacheKey);

                if (cachedMeta) {
                    const parsed = JSON.parse(cachedMeta);
                    // 첫 페이지 분량 즉시 처리 (상태images가 아닌 로컬parsed를 사용하여 의존성 루프 방지)
                    const initialBatch = parsed.slice(0, pageSize);
                    for (const img of initialBatch) {
                        const blob = await getCachedBlob(img.fullPath);
                        if (blob) {
                            img.displayUrl = URL.createObjectURL(blob);
                        } else {
                            img.displayUrl = img.url;
                        }
                    }
                    images = parsed;
                    displayedImages = initialBatch;
                    loading = false;
                }

                // 2. 백그라운드에서 Firebase 최신 데이터 가져오기
                const rootPath =
                    activeTab === "photos" ? "chat_images" : "drawings";
                const rootRef = ref(storage, rootPath);
                const allFiles = await fetchAllImages(rootRef);
                const freshImages = allFiles.sort(
                    (a, b) => b.timeCreated - a.timeCreated,
                );

                // 3. 변경사항이 있거나 캐시가 없었다면 상태 업데이트
                const isSame =
                    cachedMeta && JSON.stringify(freshImages) === cachedMeta;

                if (!isSame) {
                    images = freshImages;
                    // 로딩 스피너 종료 전에 첫 배치 데이터 준비
                    if (displayedImages.length === 0) {
                        loading = false;
                        await loadMore();
                    }
                    // 4. 새로운 메타데이터 캐싱
                    localStorage.setItem(cacheKey, JSON.stringify(freshImages));
                }
            } catch (error) {
                console.error("Gallery failed:", error);
            } finally {
                loading = false;
            }
        };

        initGallery();
    });

    function handleScroll(e) {
        const { scrollTop, scrollHeight, clientHeight } =
            e.target.scrollingElement || document.documentElement;
        if (scrollHeight - scrollTop <= clientHeight + 100) {
            loadMore();
        }
    }

    onMount(() => {
        window.addEventListener("scroll", handleScroll);
        return () => window.removeEventListener("scroll", handleScroll);
    });
</script>

<div class="gallery-container">
    <header class="gallery-header">
        <div class="tabs-container">
            <button
                class="back-btn"
                onclick={() => history.back()}
                aria-label="Go back"
            >
                <svg
                    xmlns="http://www.w3.org/2000/svg"
                    width="20"
                    height="20"
                    viewBox="0 0 24 24"
                    fill="none"
                    stroke="currentColor"
                    stroke-width="2.5"
                    stroke-linecap="round"
                    stroke-linejoin="round"
                >
                    <line x1="19" y1="12" x2="5" y2="12"></line>
                    <polyline points="12 19 5 12 12 5"></polyline>
                </svg>
            </button>
            <div class="tabs">
                <button
                    class:active={activeTab === "photos"}
                    onclick={() => (activeTab = "photos")}
                >
                    셀카 갤러리
                </button>
                <button
                    class:active={activeTab === "drawings"}
                    onclick={() => (activeTab = "drawings")}
                >
                    그림 갤러리
                </button>
            </div>
        </div>
    </header>

    <main class="gallery-content">
        {#if displayedImages.length > 0}
            <div
                class="image-grid"
                class:is-drawings={activeTab === "drawings"}
            >
                {#each displayedImages as image (image.fullPath)}
                    <button
                        class="card"
                        class:expanded={expandedId === image.fullPath}
                        onclick={() =>
                            (expandedId =
                                expandedId === image.fullPath
                                    ? null
                                    : image.fullPath)}
                        aria-expanded={expandedId === image.fullPath}
                        aria-label="Toggle larger view"
                    >
                        <div class="img-box">
                            <img
                                src={image.displayUrl || image.url}
                                alt=""
                                loading="lazy"
                            />
                        </div>
                    </button>
                {/each}
            </div>
        {/if}

        {#if loading}
            <div class="status-msg">
                <div class="spinner"></div>
                <p>로딩 중...</p>
            </div>
        {/if}

        {#if !loading && images.length > 0 && displayedImages.length >= images.length}
            <p class="end-msg">모든 이미지를 다 보셨습니다. ✨</p>
        {/if}

        {#if !loading && images.length === 0}
            <div class="empty-state">
                <p>아직 등록된 사진이 없어요.</p>
            </div>
        {/if}
    </main>

    {#if selectedImage}
        <div
            class="modal-backdrop"
            onclick={() => (selectedImage = null)}
            onkeydown={(e) => e.key === "Escape" && (selectedImage = null)}
            transition:fade={{ duration: 200 }}
            role="button"
            tabindex="0"
            aria-label="Close image modal"
        >
            <div
                class="modal-content"
                onclick={(e) => e.stopPropagation()}
                role="none"
            >
                <img
                    src={selectedImage.displayUrl || selectedImage.url}
                    alt=""
                    class="modal-image"
                />
                <button
                    class="close-btn"
                    onclick={() => (selectedImage = null)}
                    aria-label="Close modal"
                >
                    <svg
                        xmlns="http://www.w3.org/2000/svg"
                        width="24"
                        height="24"
                        viewBox="0 0 24 24"
                        fill="none"
                        stroke="currentColor"
                        stroke-width="2"
                        stroke-linecap="round"
                        stroke-linejoin="round"
                    >
                        <line x1="18" y1="6" x2="6" y2="18"></line>
                        <line x1="6" y1="6" x2="18" y2="18"></line>
                    </svg>
                </button>
            </div>
        </div>
    {/if}
</div>

<style>
    .gallery-container {
        max-width: 1200px;
        margin: 0 auto;
        padding: 4rem 1.5rem;
    }

    .gallery-header {
        text-align: center;
        margin-bottom: 4rem;
    }

    h1 {
        font-size: 2.5rem;
        font-weight: 800;
        margin-bottom: 0.5rem;
        background: linear-gradient(135deg, #333 0%, #666 100%);
        -webkit-background-clip: text;
        background-clip: text;
        -webkit-text-fill-color: transparent;
    }

    .subtitle {
        color: #888;
        font-size: 1.1rem;
        margin-bottom: 2.5rem;
    }

    .tabs-container {
        display: inline-flex;
        align-items: center;
        gap: 1rem;
    }

    .back-btn {
        width: 44px;
        height: 44px;
        border-radius: 50%;
        border: none;
        background: #f1f3f5;
        color: #555;
        cursor: pointer;
        display: flex;
        align-items: center;
        justify-content: center;
        transition: all 0.3s ease;
    }

    .back-btn:hover {
        background: #e9ecef;
        transform: translateX(-3px);
        color: #000;
    }

    .tabs {
        display: inline-flex;
        background: #f1f3f5;
        padding: 0.4rem;
        border-radius: 16px;
        gap: 0.4rem;
    }

    .tabs button {
        padding: 0.8rem 2rem;
        border: none;
        background: none;
        border-radius: 12px;
        font-size: 1rem;
        font-weight: 700;
        color: #888;
        cursor: pointer;
        transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    }

    .tabs button.active {
        background: #fff;
        color: #0066cc;
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.08);
        transform: scale(1.05);
    }

    .image-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
        gap: 2rem;
    }

    .card {
        background: #fff;
        border-radius: 20px;
        overflow: hidden;
        box-shadow: 0 10px 30px rgba(0, 0, 0, 0.05);
        transition: all 0.5s cubic-bezier(0.16, 1, 0.3, 1);
        border: none;
        padding: 0;
        cursor: pointer;
        width: 100%;
        position: relative;
    }

    .card.expanded {
        grid-column: span 2;
        grid-row: span 2;
        z-index: 10;
        transform: scale(1);
        box-shadow: 0 20px 50px rgba(0, 0, 0, 0.15);
    }

    .card:hover:not(.expanded) {
        transform: translateY(-5px) scale(1.02);
        box-shadow: 0 15px 35px rgba(0, 0, 0, 0.08);
    }

    .img-box {
        aspect-ratio: 1 / 1;
        overflow: hidden;
        background: #f8f9fa;
        display: flex;
        align-items: center;
        justify-content: center;
    }

    .image-grid.is-drawings .img-box {
        aspect-ratio: auto;
    }

    .img-box img {
        width: 100%;
        height: 100%;
        object-fit: cover;
    }

    .image-grid.is-drawings .img-box img {
        height: auto;
        max-height: 80vh;
    }

    .status-msg {
        grid-column: 1 / -1;
        text-align: center;
        padding: 4rem 0;
        color: #888;
    }

    .spinner {
        width: 30px;
        height: 30px;
        border: 3px solid #f3f3f3;
        border-top: 3px solid #0066cc;
        border-radius: 50%;
        margin: 0 auto 1rem;
        animation: spin 1s linear infinite;
    }

    .end-msg {
        grid-column: 1 / -1;
        text-align: center;
        padding: 4rem 0;
        color: #bbb;
        font-size: 0.9rem;
    }

    .empty-state {
        grid-column: 1 / -1;
        text-align: center;
        padding: 10rem 0;
        color: #ccc;
    }

    @keyframes spin {
        0% {
            transform: rotate(0deg);
        }
        100% {
            transform: rotate(360deg);
        }
    }

    @media (max-width: 768px) {
        .image-grid {
            grid-template-columns: repeat(2, 1fr);
            gap: 1rem;
        }
        .card.expanded {
            grid-column: span 2;
            grid-row: span 2;
        }
        h1 {
            font-size: 2rem;
        }
    }
</style>
