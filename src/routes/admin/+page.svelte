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
        getDocs,
        limit,
    } from "firebase/firestore";
    import { storage } from "$lib/firebase";
    import { ref, uploadBytes, getDownloadURL } from "firebase/storage";
    import { onMount } from "svelte";

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

    // Stats
    let totalStats = $state(0);
    let recentStats = $state([]);
    let statsLoading = $state(true);

    $effect(() => {
        // Fetch Games
        const gamesRef = collection(db, "Games");
        const qGames = query(gamesRef, orderBy("createdAt", "desc"));
        const unsubGames = onSnapshot(qGames, (snapshot) => {
            games = snapshot.docs.map((doc) => ({ id: doc.id, ...doc.data() }));
        });

        // Fetch Stats
        const statsRef = collection(db, "Stats");
        const qStats = query(statsRef, orderBy("timestamp", "desc"), limit(50));
        const unsubStats = onSnapshot(qStats, (snapshot) => {
            recentStats = snapshot.docs.map((doc) => ({
                id: doc.id,
                ...doc.data(),
            }));
        });

        // Get total count (simple way for now)
        getDocs(statsRef).then((snap) => {
            totalStats = snap.size;
            statsLoading = false;
        });

        return () => {
            unsubGames();
            unsubStats();
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

                    // Try AVIF, fallback to WebP
                    canvas.toBlob(
                        (blob) => {
                            if (blob) {
                                resolve(blob);
                            } else {
                                reject(new Error("Canvas toBlob failed"));
                            }
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
        window.scrollTo({ top: 0, behavior: "smooth" });
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

<div class="admin-container">
    <header class="admin-header">
        <h1>관리자 페이지</h1>
        <a href="/" class="home-link">홈으로</a>
    </header>

    <div class="admin-grid">
        <!-- Stats Section -->
        <section class="admin-section stats-card">
            <h2>접속 통계</h2>
            {#if statsLoading}
                <p>계산 중...</p>
            {:else}
                <div class="total-badge">{totalStats}</div>
                <p class="label">총 누적 방문수</p>
                <div class="stats-list">
                    <h3>최근 기록 (Top 50)</h3>
                    <div class="scroll-area">
                        <table>
                            <thead>
                                <tr>
                                    <th>경로</th>
                                    <th>시간</th>
                                </tr>
                            </thead>
                            <tbody>
                                {#each recentStats as stat}
                                    <tr>
                                        <td>{stat.path}</td>
                                        <td
                                            >{stat.timestamp
                                                ?.toDate()
                                                .toLocaleString() || "..."}</td
                                        >
                                    </tr>
                                {/each}
                            </tbody>
                        </table>
                    </div>
                </div>
            {/if}
        </section>

        <!-- Games Management Section -->
        <section class="admin-section">
            <h2>게임 컨텐츠 관리</h2>

            <form onsubmit={handleAddGame} class="game-form">
                <input
                    type="text"
                    bind:value={gameTitle}
                    placeholder="게임명"
                    required
                />
                <input
                    type="url"
                    bind:value={gameUrl}
                    placeholder="게임 URL (https://...)"
                    required
                />
                <label class="file-label">
                    <span>썸네일 이미지 업로드 (AVIF 자동변환)</span>
                    <input
                        type="file"
                        accept="image/*"
                        bind:this={fileInput}
                        onchange={(e) => (thumbnailFile = e.target.files[0])}
                    />
                </label>
                <textarea bind:value={description} placeholder="게임 설명"
                ></textarea>
                <button type="submit" disabled={isSubmitting}>
                    {isSubmitting
                        ? "처리 중..."
                        : editingId
                          ? "게임 정보 수정"
                          : "게임 추가하기"}
                </button>
                {#if editingId}
                    <button
                        type="button"
                        onclick={resetForm}
                        class="cancel-btn"
                        disabled={isSubmitting}
                    >
                        수정 취소
                    </button>
                {/if}
            </form>

            <div class="games-list">
                <h3>등록된 게임 목록</h3>
                {#each games as game (game.id)}
                    <div
                        class="game-item {editingId === game.id
                            ? 'active'
                            : ''}"
                    >
                        <div
                            class="info"
                            onclick={() => startEdit(game)}
                            role="button"
                            tabindex="0"
                            onkeypress={(e) =>
                                e.key === "Enter" && startEdit(game)}
                        >
                            <strong>{game.gameTitle}</strong>
                            <span>{game.gameUrl}</span>
                        </div>
                        <button
                            onclick={() => handleDeleteGame(game.id)}
                            class="delete-btn">삭제</button
                        >
                    </div>
                {/each}
            </div>
        </section>
    </div>
</div>

<style>
    :global(body) {
        background-color: #f4f6f8;
    }

    .admin-container {
        max-width: 1200px;
        margin: 0 auto;
        padding: 2rem;
        font-family: inherit;
    }

    .admin-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        margin-bottom: 2rem;
        padding-bottom: 1rem;
        border-bottom: 2px solid #ddd;
    }

    .admin-header h1 {
        margin: 0;
        font-size: 2rem;
    }

    .home-link {
        padding: 0.5rem 1rem;
        background: #eee;
        border-radius: 4px;
        color: #333;
    }

    .admin-grid {
        display: grid;
        grid-template-columns: 1fr 2fr;
        gap: 2rem;
    }

    @media (max-width: 900px) {
        .admin-grid {
            grid-template-columns: 1fr;
        }
    }

    .admin-section {
        background: #fff;
        padding: 2rem;
        border-radius: 12px;
        box-shadow: 0 4px 6px rgba(0, 0, 0, 0.05);
    }

    .admin-section h2 {
        margin-top: 0;
        margin-bottom: 1.5rem;
        font-size: 1.5rem;
        color: #333;
    }

    /* Stats Card */
    .total-badge {
        font-size: 4rem;
        font-weight: 800;
        color: #0066cc;
        line-height: 1;
    }

    .label {
        font-size: 1rem;
        color: #666;
        margin-bottom: 2rem;
    }

    .scroll-area {
        max-height: 400px;
        overflow-y: auto;
        border: 1px solid #eee;
        border-radius: 8px;
    }

    table {
        width: 100%;
        border-collapse: collapse;
        font-size: 0.85rem;
    }

    th,
    td {
        padding: 0.75rem;
        text-align: left;
        border-bottom: 1px solid #eee;
    }

    th {
        background: #f9f9f9;
        position: sticky;
        top: 0;
    }

    /* Form */
    .game-form {
        display: flex;
        flex-direction: column;
        gap: 0.75rem;
        margin-bottom: 2rem;
        padding-bottom: 2rem;
        border-bottom: 1px dashed #ddd;
    }

    .game-form input,
    .game-form textarea,
    .game-form .file-label {
        padding: 0.75rem;
        border: 1px solid #ddd;
        border-radius: 6px;
        font-size: 1rem;
    }

    .game-form .file-label {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
        background: #f9f9f9;
        cursor: pointer;
    }

    .game-form .file-label span {
        font-size: 0.85rem;
        color: #666;
        font-weight: 600;
    }

    .game-form input[type="file"] {
        padding: 0;
        border: none;
    }

    .game-form button {
        padding: 0.75rem;
        background: #0066cc;
        color: white;
        border: none;
        border-radius: 6px;
        cursor: pointer;
        font-weight: 600;
    }

    .game-form button:hover {
        background: #0052a3;
    }

    .game-form .cancel-btn {
        background: #f0f0f0;
        color: #333;
        border: 1px solid #ddd;
        margin-top: -0.25rem;
    }

    .game-form .cancel-btn:hover {
        background: #e6e6e6;
    }

    /* List */
    .game-item.active {
        border-color: #0066cc;
        background: #f0f7ff;
    }

    .game-item .info {
        display: flex;
        flex-direction: column;
        cursor: pointer;
        flex-grow: 1;
    }

    .delete-btn {
        background: #ff4d4d;
        color: white;
        border: none;
        padding: 0.4rem 0.8rem;
        border-radius: 4px;
        cursor: pointer;
        margin-left: 1rem;
    }

    .delete-btn:hover {
        background: #cc0000;
    }
</style>
