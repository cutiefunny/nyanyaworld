<script>
    import { db } from "$lib/firebase";
    import { collection, onSnapshot, query, orderBy } from "firebase/firestore";

    let games = $state([]);
    let loading = $state(true);

    $effect(() => {
        const gamesRef = collection(db, "Games");
        const q = query(gamesRef, orderBy("createdAt", "desc"));

        const unsubscribe = onSnapshot(q, (snapshot) => {
            games = snapshot.docs.map((doc) => ({
                id: doc.id,
                ...doc.data(),
            }));
            loading = false;
        });

        return unsubscribe;
    });
</script>

<div class="container">
    {#if loading}
        <p class="loading">로딩 중...</p>
    {:else}
        <div class="grid">
            {#each games as game (game.id)}
                {@const imageUrl = game.gameThumbnail || "/images/icon-512.png"}

                <a
                    href={game.gameUrl}
                    target="_blank"
                    rel="noopener noreferrer"
                    class="card"
                >
                    <img
                        src={imageUrl}
                        alt={game.gameTitle}
                        width="300"
                        height="300"
                    />
                    <h2>{game.gameTitle}</h2>
                </a>
            {/each}
        </div>
    {/if}

    <div class="actions">
        <a href="/" class="btn btn-secondary">홈으로 돌아가기</a>
    </div>
</div>

<style>
    .container {
        max-width: 1000px;
        margin: 0 auto;
        padding: 1rem;
    }

    .loading {
        text-align: center;
        font-size: 1.2rem;
        color: #666;
        margin: 3rem 0;
    }

    .grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(300px, 1fr));
        gap: 1rem;
        margin-bottom: 1rem;
    }

    .card {
        display: flex;
        flex-direction: column;
        align-items: center;
        text-decoration: none;
        color: #333;
        background: #fff;
        border: 1px solid #c5c5c5;
        border-radius: 12px;
        padding: 1.25rem;
        transition:
            transform 0.2s ease,
            box-shadow 0.2s ease;
    }

    .card:hover {
        transform: translateY(-5px);
        box-shadow: 0 10px 20px rgba(0, 0, 0, 0.08);
        border-color: #ddd;
    }

    .card img {
        border-radius: 8px;
        object-fit: cover;
        width: 100%;
        max-width: 300px;
        height: auto;
        aspect-ratio: 1 / 1;
    }

    .card h2 {
        margin-top: 1rem;
        font-size: 1.1rem;
        font-weight: 600;
        text-align: center;
    }

    .actions {
        text-align: center;
        margin-top: 2rem;
    }

    .btn {
        display: inline-block;
        padding: 0.75rem 2rem;
        font-size: 1rem;
        border-radius: 4px;
        font-weight: 600;
        text-decoration: none;
        transition: all 0.3s ease;
    }

    .btn-secondary {
        background-color: #f0f0f0;
        color: #000;
        border: 1px solid #ddd;
    }

    .btn-secondary:hover {
        background-color: #e6e6e6;
    }
</style>
