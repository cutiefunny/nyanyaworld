<script>
    import { db } from "$lib/firebase";
    import {
        collection,
        doc,
        getDoc,
        getDocs,
        onSnapshot,
        query,
        orderBy,
        limit,
    } from "firebase/firestore";

    let { data } = $props();
    let toonId = $derived(data.toonId);

    let comicTitle = $state("");
    let episodes = $state([]);
    let loading = $state(true);

    $effect(() => {
        if (!toonId) return;

        const fetchComicTitle = async () => {
            const comicDocRef = doc(db, "Comics", toonId);
            const comicSnap = await getDoc(comicDocRef);
            if (comicSnap.exists()) {
                comicTitle = comicSnap.data().title;
            } else {
                console.error("해당 ID의 만화를 찾을 수 없습니다.");
                comicTitle = "알 수 없는 만화";
            }
        };

        fetchComicTitle();

        const episodesCollectionRef = collection(
            db,
            "Comics",
            toonId,
            "Episodes",
        );
        const q = query(episodesCollectionRef, orderBy("uploadDate", "asc"));

        const unsubscribe = onSnapshot(q, async (querySnapshot) => {
            loading = true;
            const episodesData = querySnapshot.docs.map((doc) => ({
                id: doc.id,
                ...doc.data(),
                thumbnailUrl: null,
            }));

            const episodesWithThumbnails = await Promise.all(
                episodesData.map(async (episode) => {
                    const imagesRef = collection(
                        db,
                        "Comics",
                        toonId,
                        "Episodes",
                        episode.id,
                        "Images",
                    );
                    const firstImageQuery = query(
                        imagesRef,
                        orderBy("order"),
                        limit(1),
                    );
                    const snapshot = await getDocs(firstImageQuery);
                    if (!snapshot.empty) {
                        episode.thumbnailUrl = snapshot.docs[0].data().imageUrl;
                    }
                    return episode;
                }),
            );

            episodes = episodesWithThumbnails.reverse();
            loading = false;
        });

        return () => unsubscribe();
    });

    let firstEpisodeId = $derived(
        episodes.length > 0 ? episodes[episodes.length - 1].id : null,
    );
</script>

<div class="container">
    <div class="episode-header">
        <a
            href="/toons"
            class="back-button"
            aria-label="만화 목록으로 돌아가기"
        >
            <svg
                stroke="currentColor"
                fill="currentColor"
                stroke-width="0"
                viewBox="0 0 448 512"
                height="1em"
                width="1em"
                xmlns="http://www.w3.org/2000/svg"
                ><path
                    d="M9.4 233.4c-12.5 12.5-12.5 32.8 0 45.3l160 160c12.5 12.5 32.8 12.5 45.3 0s12.5-32.8 0-45.3L109.2 288 416 288c17.7 0 32-14.3 32-32s-14.3-32-32-32l-306.7 0L214.6 118.6c12.5-12.5 12.5-32.8 0-45.3s-32.8-12.5-45.3 0l-160 160z"
                ></path></svg
            >
        </a>
        <h1 class="title">{comicTitle}</h1>
    </div>

    {#if loading}
        <p class="loading">로딩 중...</p>
    {:else}
        <div class="list">
            {#if firstEpisodeId}
                <a
                    href={`/toons/${toonId}/${firstEpisodeId}`}
                    class="list-item first-episode-button"
                >
                    <div class="episode-info">
                        <svg
                            stroke="currentColor"
                            fill="currentColor"
                            stroke-width="0"
                            viewBox="0 0 512 512"
                            height="1em"
                            width="1em"
                            xmlns="http://www.w3.org/2000/svg"
                            ><path
                                d="M256 8C119 8 8 119 8 256s111 248 248 248 248-111 248-248S393 8 256 8zm115.7 272l-176 101c-15.8 8.8-35.7-2.5-35.7-21V152c0-18.4 19.8-29.8 35.7-21l176 107c16.4 9.2 16.4 32.9 0 42z"
                            ></path></svg
                        >
                        <span>처음부터 보기</span>
                    </div>
                </a>
            {/if}

            {#each episodes as episode (episode.id)}
                <a href={`/toons/${toonId}/${episode.id}`} class="list-item">
                    <div class="episode-info">
                        <span>{episode.episodeTitle}</span>
                    </div>
                    {#if episode.thumbnailUrl}
                        <img
                            src={episode.thumbnailUrl}
                            alt={`${episode.episodeTitle} 썸네일`}
                            width="80"
                            height="50"
                            class="episode-thumbnail"
                        />
                    {/if}
                </a>
            {/each}
        </div>
    {/if}
</div>

<style>
    .container {
        max-width: 800px;
        margin: 0 auto;
        padding: 2rem;
    }

    .episode-header {
        display: flex;
        align-items: center;
        margin-bottom: 2rem;
    }

    .back-button {
        background: none;
        border: none;
        font-size: 1.5rem;
        cursor: pointer;
        color: #333;
        padding: 0.5rem;
        display: flex;
        align-items: center;
        text-decoration: none;
    }

    .back-button:hover {
        color: #0066cc;
    }

    .title {
        font-size: 1.8rem;
        margin-left: 1rem;
        color: #000;
    }

    .list {
        display: flex;
        flex-direction: column;
        gap: 1rem;
    }

    .list-item {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 1rem 1.5rem;
        background: #fff;
        border: 1px solid #eaeaea;
        border-radius: 8px;
        text-decoration: none;
        color: #333;
        transition: all 0.2s ease;
    }

    .list-item:hover {
        box-shadow: 0 4px 12px rgba(0, 0, 0, 0.05);
        border-color: #ddd;
    }

    .first-episode-button {
        background: #0066cc;
        color: white;
        border: none;
    }

    .first-episode-button:hover {
        background: #0052a3;
        color: white;
    }

    .first-episode-button .episode-info svg {
        margin-right: 0.5rem;
        font-size: 1.2rem;
        transform: translateY(2px);
    }

    .episode-info {
        display: flex;
        align-items: center;
        font-weight: 600;
        font-size: 1.1rem;
    }

    .episode-thumbnail {
        border-radius: 4px;
        object-fit: cover;
    }

    .loading {
        text-align: center;
        font-size: 1.2rem;
        color: #666;
        margin: 3rem 0;
    }
</style>
