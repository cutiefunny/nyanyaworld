<script>
    import { db } from "$lib/firebase";
    import { collection, onSnapshot, query, orderBy } from "firebase/firestore";

    let comics = $state([]);
    let loading = $state(true);

    $effect(() => {
        const comicsCollectionRef = collection(db, "Comics");
        const q = query(comicsCollectionRef, orderBy("order", "asc"));

        const unsubscribe = onSnapshot(q, (querySnapshot) => {
            const comicsData = [];
            querySnapshot.forEach((doc) => {
                comicsData.push({ id: doc.id, ...doc.data() });
            });
            comics = comicsData;
            loading = false;
        });

        return unsubscribe;
    });
</script>

<div class="container">
    {#if loading}
        <p class="loading">로딩 중...</p>
    {:else}
        <h1 class="title">만화 목록</h1>
        <div class="grid">
            {#each comics as comic (comic.id)}
                {@const imageUrl =
                    comic.thumbnailUrl &&
                    (comic.thumbnailUrl.startsWith("http") ||
                        comic.thumbnailUrl.startsWith("/"))
                        ? comic.thumbnailUrl
                        : "/images/icon-512.png"}

                <a href={`/toons/${comic.id}`} class="card">
                    <img
                        src={imageUrl}
                        alt={comic.title}
                        width="300"
                        height="300"
                    />
                    <h2>{comic.title}</h2>
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
        padding: 2rem;
    }

    .title {
        text-align: center;
        font-size: 2.5rem;
        margin-bottom: 2rem;
        color: #000;
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
        border: 1px solid #eaeaea;
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
