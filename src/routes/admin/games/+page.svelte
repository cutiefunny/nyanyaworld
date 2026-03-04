<script>
    import { db } from "$lib/firebase";
    import {
        collection,
        addDoc,
        onSnapshot,
        query,
        orderBy,
        updateDoc,
        deleteDoc,
        doc,
        serverTimestamp,
    } from "firebase/firestore";
    import { storage } from "$lib/firebase";
    import { ref, uploadBytes, getDownloadURL } from "firebase/storage";

    // Games Management
    let games = $state([]);
    let gameTitle = $state("");
    let gameUrl = $state("");
    let gameThumbnail = $state("");
    let description = $state("");
    let isSubmitting = $state(false);
    let thumbnailFile = $state(null);
    let editingId = $state(null);
    let fileInput;

    $effect(() => {
        // Fetch Games
        const gamesRef = collection(db, "Games");
        const qGames = query(gamesRef, orderBy("createdAt", "desc"));
        const unsubGames = onSnapshot(qGames, (snapshot) => {
            games = snapshot.docs.map((doc) => ({ id: doc.id, ...doc.data() }));
        });

        return () => {
            unsubGames();
        };
    });

    async function processImage(file) {
        return new Promise((resolve, reject) => {
            const reader = new FileReader();
            reader.onload = (e) => {
                const img = new Image();
                img.onload = () => {
                    const canvas = document.createElement("canvas");
                    let width = img.width;
                    let height = img.height;
                    const maxDim = 800;

                    if (width > height) {
                        if (width > maxDim) {
                            height *= maxDim / width;
                            width = maxDim;
                        }
                    } else {
                        if (height > maxDim) {
                            width *= maxDim / height;
                            height = maxDim;
                        }
                    }

                    canvas.width = width;
                    canvas.height = height;
                    const ctx = canvas.getContext("2d");
                    ctx.drawImage(img, 0, 0, width, height);

                    canvas.toBlob(
                        (blob) => {
                            if (blob) resolve(blob);
                            else reject(new Error("Canvas toBlob failed"));
                        },
                        "image/avif",
                        0.7,
                    );
                };
                img.onerror = reject;
                img.src = e.target.result;
            };
            reader.onerror = reject;
            reader.readAsDataURL(file);
        });
    }

    async function handleAddGame(e) {
        e.preventDefault();
        if (!gameTitle || !gameUrl) return;

        isSubmitting = true;
        try {
            let uploadedThumbnailUrl = gameThumbnail;

            if (thumbnailFile) {
                const processedBlob = await processImage(thumbnailFile);
                const fileName = `games/${Date.now()}_${gameTitle.substring(0, 10)}.avif`;
                const storageRef = ref(storage, fileName);
                const uploadResult = await uploadBytes(
                    storageRef,
                    processedBlob,
                );
                uploadedThumbnailUrl = await getDownloadURL(uploadResult.ref);
            }

            const gameData = {
                gameTitle,
                gameUrl,
                gameThumbnail: uploadedThumbnailUrl,
                description,
            };

            if (editingId) {
                await updateDoc(doc(db, "Games", editingId), gameData);
                alert("게임 정보가 수정되었습니다.");
            } else {
                await addDoc(collection(db, "Games"), {
                    ...gameData,
                    createdAt: serverTimestamp(),
                });
                alert("게임이 등록되었습니다.");
            }

            resetForm();
        } catch (error) {
            console.error(error);
            alert("작업 실패: " + error.message);
        } finally {
            isSubmitting = false;
        }
    }

    function resetForm() {
        gameTitle = "";
        gameUrl = "";
        gameThumbnail = "";
        description = "";
        thumbnailFile = null;
        editingId = null;
        if (fileInput) fileInput.value = "";
    }

    function startEdit(game) {
        gameTitle = game.gameTitle;
        gameUrl = game.gameUrl;
        gameThumbnail = game.gameThumbnail;
        description = game.description || "";
        editingId = game.id;
        thumbnailFile = null;
        if (fileInput) fileInput.value = "";
    }

    async function handleDeleteGame(id) {
        if (!confirm("정말 삭제하시겠습니까?")) return;
        try {
            await deleteDoc(doc(db, "Games", id));
        } catch (error) {
            console.error(error);
        }
    }
</script>

<section class="admin-section">
    <header class="section-header">
        <h2>게임 컨텐츠 관리</h2>
    </header>

    <div class="management-grid">
        <!-- Form Area -->
        <div class="form-card">
            <h3>{editingId ? "게임 정보 수정" : "새 게임 등록"}</h3>
            <form onsubmit={handleAddGame} class="game-form">
                <div class="ip-group">
                    <label>게임 제목</label>
                    <input
                        type="text"
                        bind:value={gameTitle}
                        placeholder="예: 냐냐 슬라이드"
                        required
                    />
                </div>
                <div class="ip-group">
                    <label>게임 실행 URL</label>
                    <input
                        type="url"
                        bind:value={gameUrl}
                        placeholder="https://..."
                        required
                    />
                </div>
                <div class="ip-group">
                    <label
                        >썸네일 이미지 {editingId
                            ? "(변경시에만 첨부)"
                            : ""}</label
                    >
                    <label class="file-label">
                        <input
                            type="file"
                            accept="image/*"
                            bind:this={fileInput}
                            onchange={(e) =>
                                (thumbnailFile = e.target.files[0])}
                        />
                        <span class="hint">AVIF 800px로 자동 최적화됩니다.</span
                        >
                    </label>
                </div>
                <div class="ip-group">
                    <label>게임 설명</label>
                    <textarea
                        bind:value={description}
                        placeholder="게임에 대한 간단한 설명을 입력하세요."
                    ></textarea>
                </div>

                <div class="btn-group">
                    <button
                        type="submit"
                        class="submit-btn"
                        disabled={isSubmitting}
                    >
                        {isSubmitting
                            ? "처리 중..."
                            : editingId
                              ? "수정 완료"
                              : "등록하기"}
                    </button>
                    {#if editingId}
                        <button
                            type="button"
                            class="cancel-btn"
                            onclick={resetForm}>취소</button
                        >
                    {/if}
                </div>
            </form>
        </div>

        <!-- List Area -->
        <div class="list-card">
            <h3>등록된 게임 목록 ({games.length})</h3>
            <div class="games-list-scroll">
                {#each games as game (game.id)}
                    <div class="game-row" class:active={editingId === game.id}>
                        <div class="game-thumb">
                            {#if game.gameThumbnail}
                                <img src={game.gameThumbnail} alt="" />
                            {:else}
                                <div class="no-img">No Img</div>
                            {/if}
                        </div>
                        <div
                            class="game-info-cell"
                            onclick={() => startEdit(game)}
                            role="button"
                            tabindex="0"
                        >
                            <div class="title-row">
                                <strong>{game.gameTitle}</strong>
                                {#if editingId === game.id}<span
                                        class="edit-badge">Editing</span
                                    >{/if}
                            </div>
                            <span class="url-text">{game.gameUrl}</span>
                        </div>
                        <button
                            class="delete-icon"
                            onclick={() => handleDeleteGame(game.id)}
                            title="삭제"
                        >
                            🗑️
                        </button>
                    </div>
                {/each}
            </div>
        </div>
    </div>
</section>

<style>
    .admin-section {
        max-width: 1200px;
        margin: 0 auto;
    }

    .section-header {
        margin-bottom: 2rem;
    }

    .section-header h2 {
        font-size: 1.75rem;
        color: #333;
        margin: 0;
    }

    /* Management Grid */
    .management-grid {
        display: grid;
        grid-template-columns: 1fr 1.5fr;
        gap: 2rem;
        align-items: start;
    }

    .form-card,
    .list-card {
        background: #fff;
        padding: 2rem;
        border-radius: 16px;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
    }

    h3 {
        margin-top: 0;
        margin-bottom: 1.5rem;
        font-size: 1.2rem;
        color: #333;
    }

    .ip-group {
        margin-bottom: 1.25rem;
    }

    .ip-group label {
        display: block;
        font-size: 0.85rem;
        font-weight: 600;
        color: #666;
        margin-bottom: 0.5rem;
    }

    input,
    textarea {
        width: 100%;
        padding: 0.75rem 1rem;
        border: 1px solid #e0e0e0;
        border-radius: 8px;
        font-size: 0.95rem;
        transition: border-color 0.2s;
    }

    input:focus,
    textarea:focus {
        outline: none;
        border-color: #0066cc;
    }

    textarea {
        height: 100px;
        resize: vertical;
    }

    .file-label {
        display: block;
        padding: 1rem;
        background: #f8f9fa;
        border: 2px dashed #e0e0e0;
        border-radius: 8px;
        cursor: pointer;
        text-align: center;
    }

    .file-label .hint {
        display: block;
        font-size: 0.75rem;
        color: #999;
        margin-top: 0.5rem;
    }

    .btn-group {
        display: flex;
        gap: 0.75rem;
        margin-top: 2rem;
    }

    .submit-btn {
        flex-grow: 1;
        padding: 1rem;
        background: #0066cc;
        color: #fff;
        border: none;
        border-radius: 8px;
        font-weight: 700;
        cursor: pointer;
    }

    .cancel-btn {
        padding: 1rem 1.5rem;
        background: #f1f3f5;
        color: #495057;
        border: none;
        border-radius: 8px;
        font-weight: 600;
        cursor: pointer;
    }

    /* List Rows */
    .games-list-scroll {
        max-height: 600px;
        overflow-y: auto;
    }

    .game-row {
        display: flex;
        align-items: center;
        padding: 1rem;
        border: 1px solid #f0f0f0;
        border-radius: 12px;
        margin-bottom: 0.75rem;
        gap: 1rem;
        transition: all 0.2s;
    }

    .game-row:hover {
        border-color: #d0d0d0;
        background: #f8f9fa;
    }

    .game-row.active {
        border-color: #0066cc;
        background: #f0f7ff;
    }

    .game-thumb {
        width: 50px;
        height: 50px;
        background: #eee;
        border-radius: 8px;
        overflow: hidden;
        flex-shrink: 0;
    }

    .game-thumb img {
        width: 100%;
        height: 100%;
        object-fit: cover;
    }

    .game-thumb .no-img {
        font-size: 0.6rem;
        color: #999;
        display: flex;
        align-items: center;
        justify-content: center;
        height: 100%;
    }

    .game-info-cell {
        flex-grow: 1;
        cursor: pointer;
    }

    .title-row {
        display: flex;
        align-items: center;
        gap: 0.5rem;
        margin-bottom: 0.25rem;
    }

    .edit-badge {
        font-size: 0.65rem;
        background: #0066cc;
        color: #fff;
        padding: 0.1rem 0.4rem;
        border-radius: 4px;
        font-weight: 800;
    }

    .url-text {
        font-size: 0.75rem;
        color: #999;
        display: block;
        max-width: 250px;
        overflow: hidden;
        text-overflow: ellipsis;
        white-space: nowrap;
    }

    .delete-icon {
        background: none;
        border: none;
        cursor: pointer;
        font-size: 1.1rem;
        opacity: 0.3;
        transition: opacity 0.2s;
    }

    .delete-icon:hover {
        opacity: 1;
    }
</style>
