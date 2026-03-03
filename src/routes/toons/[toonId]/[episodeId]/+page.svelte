<script>
    import { db } from "$lib/firebase";
    import {
        collection,
        getDocs,
        onSnapshot,
        query,
        orderBy,
    } from "firebase/firestore";
    import emblaCarouselSvelte from "embla-carousel-svelte";

    let { data } = $props();
    let toonId = $derived(data.toonId);
    let episodeId = $derived(decodeURIComponent(data.episodeId));

    let images = $state([]);
    let loading = $state(true);
    let prevEpisodeId = $state(null);
    let nextEpisodeId = $state(null);
    let currentIndex = $state(0);
    let emblaApi = $state(null);

    const emblaOptions = { loop: false, duration: 25 };

    function onInit(event) {
        emblaApi = event.detail;
        emblaApi.on("select", () => {
            currentIndex = emblaApi.selectedScrollSnap();
        });
    }

    function handlePrevCut() {
        if (emblaApi && emblaApi.canScrollPrev()) {
            emblaApi.scrollPrev();
        } else if (prevEpisodeId) {
            window.location.href = `/toons/${toonId}/${prevEpisodeId}`;
        }
    }

    function handleNextCut() {
        if (emblaApi && emblaApi.canScrollNext()) {
            emblaApi.scrollNext();
        } else if (nextEpisodeId) {
            window.location.href = `/toons/${toonId}/${nextEpisodeId}`;
        }
    }

    $effect(() => {
        if (!toonId || !episodeId) return;

        let unsubscribe;

        const fetchData = async () => {
            loading = true;
            try {
                const episodesQuery = query(
                    collection(db, `Comics/${toonId}/Episodes`),
                    orderBy("uploadDate", "asc"),
                );
                const episodesSnapshot = await getDocs(episodesQuery);
                const allEpisodes = episodesSnapshot.docs.map((doc) => doc.id);

                const foundIndex = allEpisodes.findIndex(
                    (id) => id === episodeId,
                );

                prevEpisodeId =
                    foundIndex > 0 ? allEpisodes[foundIndex - 1] : null;
                nextEpisodeId =
                    foundIndex < allEpisodes.length - 1
                        ? allEpisodes[foundIndex + 1]
                        : null;

                const imagesCollectionRef = collection(
                    db,
                    "Comics",
                    toonId,
                    "Episodes",
                    episodeId,
                    "Images",
                );
                const q = query(imagesCollectionRef, orderBy("order"));

                unsubscribe = onSnapshot(q, (querySnapshot) => {
                    images = querySnapshot.docs.map((doc) => ({
                        id: doc.id,
                        ...doc.data(),
                    }));
                    loading = false;
                });
            } catch (error) {
                console.error("데이터를 가져오는 중 오류 발생:", error);
                loading = false;
            }
        };

        fetchData();

        return () => {
            if (unsubscribe) unsubscribe();
        };
    });
</script>

<div class="viewer-container">
    {#if loading}
        <p class="loading">로딩 중...</p>
    {:else}
        <nav class="viewer-nav">
            <a href={`/toons/${toonId}`} class="nav-btn list-btn">
                <svg
                    stroke="currentColor"
                    fill="currentColor"
                    stroke-width="0"
                    viewBox="0 0 448 512"
                    height="1em"
                    width="1em"
                    xmlns="http://www.w3.org/2000/svg"
                    style="vertical-align: middle; margin-right: 0.25rem;"
                    ><path
                        d="M9.4 233.4c-12.5 12.5-12.5 32.8 0 45.3l160 160c12.5 12.5 32.8 12.5 45.3 0s12.5-32.8 0-45.3L109.2 288 416 288c17.7 0 32-14.3 32-32s-14.3-32-32-32l-306.7 0L214.6 118.6c12.5-12.5 12.5-32.8 0-45.3s-32.8-12.5-45.3 0l-160 160z"
                    ></path></svg
                >
                목록으로
            </a>
            <div class="nav-controls">
                <button
                    class="nav-btn {!prevEpisodeId && currentIndex === 0
                        ? 'disabled'
                        : ''}"
                    onclick={handlePrevCut}>이전</button
                >
                <button
                    class="nav-btn {!nextEpisodeId &&
                    currentIndex === images.length - 1
                        ? 'disabled'
                        : ''}"
                    onclick={handleNextCut}>다음</button
                >
            </div>
        </nav>

        <div class="slider">
            {#if images.length === 0}
                <p class="no-images">이미지가 없습니다.</p>
            {:else}
                <div
                    class="embla"
                    use:emblaCarouselSvelte={{ options: emblaOptions }}
                    onemblaInit={onInit}
                >
                    <div class="embla__container">
                        {#each images as image (image.id)}
                            <div class="embla__slide">
                                <img
                                    src={image.imageUrl}
                                    alt={`이미지 컷 ${image.order}`}
                                    class="comic-image"
                                />
                            </div>
                        {/each}
                    </div>

                    <!-- PC용 화살표 -->
                    <button
                        class="slide-nav left-nav"
                        aria-label="이전 페이지"
                        onclick={handlePrevCut}
                    >
                        <span class="arrow-icon">&#10094;</span>
                    </button>
                    <button
                        class="slide-nav right-nav"
                        aria-label="다음 페이지"
                        onclick={handleNextCut}
                    >
                        <span class="arrow-icon">&#10095;</span>
                    </button>
                </div>

                <div class="slide-indicator">
                    {currentIndex + 1} / {images.length}
                </div>
            {/if}
        </div>
    {/if}
</div>

<style>
    :global(body) {
        background-color: #ffffff;
        overflow-x: hidden;
    }

    .viewer-container {
        max-width: 100%;
        height: 100vh;
        display: flex;
        flex-direction: column;
        background: #ffffff;
        color: #333333;
    }

    .loading {
        text-align: center;
        padding: 5rem 0;
        font-size: 1.2rem;
        color: #999;
    }

    .viewer-nav {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 0.75rem 1rem;
        background: #ffffff;
        z-index: 10;
        border-bottom: 1px solid #eeeeee;
    }

    .nav-controls {
        display: flex;
        gap: 0.5rem;
    }

    .nav-btn {
        color: #333333;
        text-decoration: none;
        padding: 0.4rem 0.8rem;
        background: #f5f5f5;
        border: 1px solid #dddddd;
        border-radius: 4px;
        font-size: 0.85rem;
        cursor: pointer;
        transition: all 0.2s ease;
    }

    .nav-btn:hover:not(.disabled) {
        background: #eeeeee;
    }

    .nav-btn.disabled {
        color: #cccccc;
        cursor: not-allowed;
    }

    .slider {
        flex: 1;
        display: flex;
        flex-direction: column;
        justify-content: center;
        position: relative;
        overflow: hidden;
    }

    .embla {
        overflow: hidden;
        width: 100%;
        height: 80vh;
        position: relative;
    }

    .embla__container {
        display: flex;
        height: 100%;
    }

    .embla__slide {
        flex: 0 0 100%;
        min-width: 0;
        display: flex;
        justify-content: center;
        align-items: center;
        padding: 0 10px;
    }

    .comic-image {
        max-width: 100%;
        max-height: 100%;
        object-fit: contain;
        user-select: none;
        -webkit-user-drag: none;
    }

    .slide-nav {
        position: absolute;
        top: 0;
        bottom: 0;
        width: 10%;
        background: transparent;
        border: none;
        cursor: pointer;
        z-index: 5;
        display: flex;
        align-items: center;
        justify-content: center;
        outline: none;
        transition: background 0.2s ease;
    }

    .slide-nav:hover {
        background: rgba(0, 0, 0, 0.03);
    }

    .left-nav {
        left: 0;
    }
    .right-nav {
        right: 0;
    }

    .arrow-icon {
        display: none;
        font-size: 3rem;
        color: rgba(0, 0, 0, 0.3);
        text-shadow: 0 0 10px rgba(255, 255, 255, 0.8);
        user-select: none;
    }

    @media (pointer: fine) {
        .arrow-icon {
            display: block;
        }

        .slide-nav:hover .arrow-icon {
            color: #000;
            transform: scale(1.1);
        }
    }

    .slide-indicator {
        margin-top: 1rem;
        background: rgba(0, 0, 0, 0.05);
        color: #666666;
        padding: 0.3rem 1rem;
        border-radius: 20px;
        font-size: 0.9rem;
        font-weight: bold;
        align-self: center;
    }

    .no-images {
        text-align: center;
        color: #999;
    }
</style>
