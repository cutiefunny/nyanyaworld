<script>
	import { db, storage } from "$lib/firebase";
	import { collection, onSnapshot, query, limit } from "firebase/firestore";
	import {
		ref,
		listAll,
		getDownloadURL,
		getMetadata,
	} from "firebase/storage";

	let records = $state([]);
	let randomComicThumbnail = $state(null);
	let randomGameThumbnail = $state(null);
	let randomGalleryThumbnail = $state(null);
	let latestGalleryImages = $state([]);
	let currentSlide = $state(0);
	let startX = $state(0);
	let isDragging = $state(false);

	// 슬라이드 타이머
	let slideInterval;
	const startTimer = () => {
		clearInterval(slideInterval);
		slideInterval = setInterval(() => {
			if (latestGalleryImages.length > 0 && !isDragging) {
				currentSlide = (currentSlide + 1) % latestGalleryImages.length;
			}
		}, 5000);
	};

	// 스와이프 핸들러
	const handleStart = (e) => {
		isDragging = true;
		startX = e.type.includes("touch") ? e.touches[0].clientX : e.clientX;
	};

	const handleEnd = (e) => {
		if (!isDragging) return;
		const endX = e.type.includes("touch")
			? e.changedTouches[0].clientX
			: e.clientX;
		const diff = startX - endX;

		if (Math.abs(diff) > 50) {
			// 최소 스와이프 거리
			if (diff > 0) {
				currentSlide = (currentSlide + 1) % latestGalleryImages.length;
			} else {
				currentSlide =
					(currentSlide - 1 + latestGalleryImages.length) %
					latestGalleryImages.length;
			}
			startTimer(); // 조작 시 타이머 리셋
		}
		isDragging = false;
	};

	$effect(() => {
		const q = query(collection(db, "records"), limit(10));
		const unsubscribeRecords = onSnapshot(q, (snapshot) => {
			records = snapshot.docs.map((doc) => ({
				id: doc.id,
				...doc.data(),
			}));
		});

		const comicsQ = query(collection(db, "Comics"));
		const unsubscribeComics = onSnapshot(comicsQ, (snapshot) => {
			const allComics = snapshot.docs.map((doc) => doc.data());
			const thumbs = allComics.map((c) => c.thumbnailUrl).filter(Boolean);

			if (thumbs.length > 0) {
				randomComicThumbnail =
					thumbs[Math.floor(Math.random() * thumbs.length)];
			}
		});

		const gamesQ = query(collection(db, "Games"));
		const unsubscribeGames = onSnapshot(gamesQ, (snapshot) => {
			const allGames = snapshot.docs.map((doc) => doc.data());
			const thumbs = allGames.map((g) => g.gameThumbnail).filter(Boolean);

			if (thumbs.length > 0) {
				randomGameThumbnail =
					thumbs[Math.floor(Math.random() * thumbs.length)];
			}
		});

		// --- IndexedDB Cache (Simplified for Home) ---
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
				console.warn("Cache fail", e);
			}
		}

		// 갤러리 썸네일 랜덤 추출 (최근 3개 중 1개)
		const fetchGalleryThumbs = async () => {
			try {
				// 1. 메타데이터 캐시 확인
				const cacheKey = "home_gallery_meta";
				const cachedMetaStr = localStorage.getItem(cacheKey);
				if (cachedMetaStr) {
					const cached = JSON.parse(cachedMetaStr);
					const randomPick =
						cached[Math.floor(Math.random() * cached.length)];
					// 블롭 캐시 확인
					const blob = await getCachedBlob(randomPick.fullPath);
					if (blob) {
						randomGalleryThumbnail = URL.createObjectURL(blob);
					} else {
						randomGalleryThumbnail = randomPick.url;
					}
				}

				const fetchAllFiles = async (dirRef) => {
					const res = await listAll(dirRef);
					const files = await Promise.all(
						res.items.map(async (item) => {
							const meta = await getMetadata(item);
							if (meta.customMetadata?.hidden === "true")
								return null;
							return {
								url: "", // URL은 필요할 때만 가져옴 (비용 절약)
								fullPath: item.fullPath,
								timeCreated: new Date(
									meta.timeCreated,
								).getTime(),
							};
						}),
					);

					const subFolderFiles = await Promise.all(
						res.prefixes.map((folder) => fetchAllFiles(folder)),
					);

					return [...files.filter(Boolean), ...subFolderFiles.flat()];
				};

				const chatImagesRef = ref(storage, "chat_images");
				const drawingsRef = ref(storage, "drawings");

				const [chatFiles, drawingFiles] = await Promise.all([
					fetchAllFiles(chatImagesRef),
					fetchAllFiles(drawingsRef),
				]);

				const allFiles = [...chatFiles, ...drawingFiles];

				if (allFiles.length > 0) {
					// 전체 최신순 정렬 후 상위 6개 추출 (충분한 풀 확보)
					const latestMeta = allFiles
						.sort((a, b) => b.timeCreated - a.timeCreated)
						.slice(0, 6);

					// 메타데이터 캐시 업데이트
					// 백그라운드에서 실제 URL 가져오기
					const latestWithUrl = await Promise.all(
						latestMeta.map(async (m) => {
							const url = await getDownloadURL(
								ref(storage, m.fullPath),
							);
							return { ...m, url };
						}),
					);

					localStorage.setItem(
						cacheKey,
						JSON.stringify(latestWithUrl),
					);

					// 슬라이드용 이미지 준비
					const blobPromises = latestWithUrl.map(async (img) => {
						const blob = await getCachedBlob(img.fullPath);
						if (blob) {
							return URL.createObjectURL(blob);
						} else {
							// 캐시가 없으면 다운로드 후 저장
							try {
								const response = await fetch(img.url);
								const newBlob = await response.blob();
								await setCachedBlob(img.fullPath, newBlob);
								return URL.createObjectURL(newBlob);
							} catch {
								return img.url;
							}
						}
					});

					latestGalleryImages = await Promise.all(blobPromises);

					// 하단 섹션용 배경 (첫번째 이미지 활용)
					if (latestGalleryImages.length > 0) {
						randomGalleryThumbnail = latestGalleryImages[0];
					}
				}
			} catch (e) {
				console.warn("Home gallery thumb failed:", e);
			}
		};

		fetchGalleryThumbs().then(startTimer);

		return () => {
			unsubscribeRecords();
			unsubscribeComics();
			unsubscribeGames();
			clearInterval(slideInterval);
		};
	});
</script>

<div class="container">
	<section class="hero">
		{#if latestGalleryImages.length > 0}
			<div
				class="carousel-container"
				onmousedown={handleStart}
				onmouseup={handleEnd}
				ontouchstart={handleStart}
				ontouchend={handleEnd}
				role="region"
				aria-label="최신 소식 슬라이더"
			>
				<div class="carousel-track">
					{#each latestGalleryImages as img, i}
						{@const diff =
							(i - currentSlide + latestGalleryImages.length) %
							latestGalleryImages.length}
						{@const position =
							diff === 0
								? "center"
								: diff === 1
									? "right"
									: "left"}
						<div class="carousel-slide {position}">
							<img src={img} alt="최신 소식" />
						</div>
					{/each}
				</div>
				<div class="carousel-dots">
					{#each latestGalleryImages as _, i}
						<div
							class="dot"
							class:active={currentSlide === i}
						></div>
					{/each}
				</div>
			</div>
		{:else}
			<h1>(최신 소식 슬라이드)</h1>
		{/if}
	</section>

	<section class="features">
		<a href="/gallery" class="feature-card">
			<div class="card-bg">
				{#if randomGalleryThumbnail}
					<img src={randomGalleryThumbnail} alt="갤러리 썸네일" />
				{:else}
					<div class="gradient-bg gallery-gradient"></div>
				{/if}
			</div>
			<div class="card-content">
				<h2>잡화점 갤러리</h2>
			</div>
		</a>

		<a href="/shop" class="feature-card">
			<div class="card-bg">
				<div class="gradient-bg shop-gradient"></div>
			</div>
			<div class="card-content">
				<h2>냐냐 쇼핑몰</h2>
			</div>
		</a>

		<a href="/toons" class="feature-card">
			{#if randomComicThumbnail}
				<div class="card-bg">
					<img src={randomComicThumbnail} alt="만화 썸네일" />
				</div>
			{/if}
			<div class="card-content">
				<h2>냐냐 코믹스</h2>
			</div>
		</a>

		<a href="/games" class="feature-card">
			<div class="card-bg">
				{#if randomGameThumbnail}
					<img src={randomGameThumbnail} alt="게임 썸네일" />
				{:else}
					<div class="gradient-bg games-gradient"></div>
				{/if}
			</div>
			<div class="card-content">
				<h2>냐냐 게임즈</h2>
			</div>
		</a>
	</section>
</div>

<style>
	.container {
		max-width: 1200px;
		margin: 0 auto;
		padding: 0 2rem;
	}

	.hero {
		text-align: center;
		padding: 4rem 0;
		overflow: hidden;
	}

	.carousel-container {
		position: relative;
		width: 100%;
		height: 480px;
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		cursor: grab;
		user-select: none;
	}

	.carousel-container:active {
		cursor: grabbing;
	}

	.carousel-track {
		position: relative;
		width: 100%;
		height: 400px;
		perspective: 1000px;
	}

	.carousel-slide {
		position: absolute;
		top: 50%;
		left: 50%;
		width: 60%;
		height: 100%;
		transform: translate(-50%, -50%);
		transition: all 0.8s cubic-bezier(0.25, 1, 0.5, 1);
		border-radius: 20px;
		overflow: hidden;
		box-shadow: 0 10px 40px rgba(0, 0, 0, 0.2);
		opacity: 0;
		z-index: 1;
	}

	.carousel-slide img {
		width: 100%;
		height: 100%;
		object-fit: cover;
	}

	.carousel-slide.center {
		opacity: 1;
		z-index: 10;
		transform: translate(-50%, -50%) scale(1.1);
		box-shadow: 0 20px 60px rgba(0, 0, 0, 0.4);
	}

	.carousel-slide.left {
		opacity: 0.6;
		z-index: 5;
		transform: translate(-95%, -50%) scale(0.85);
		filter: blur(2px);
	}

	.carousel-slide.right {
		opacity: 0.6;
		z-index: 5;
		transform: translate(-5%, -50%) scale(0.85);
		filter: blur(2px);
	}

	.carousel-dots {
		display: flex;
		gap: 0.8rem;
		margin-top: 2rem;
	}

	.dot {
		width: 8px;
		height: 8px;
		border-radius: 50%;
		background: #ddd;
		transition: all 0.3s ease;
	}

	.dot.active {
		background: #4facfe;
		width: 24px;
		border-radius: 4px;
	}

	.features {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
		gap: 2rem;
		padding: 3rem 0;
	}

	.feature-card {
		position: relative;
		aspect-ratio: 1 / 1;
		display: flex;
		flex-direction: column;
		justify-content: center;
		align-items: center;
		overflow: hidden;
		border-radius: 12px;
		background-color: #fafafa;
		transition: all 0.3s ease;
		text-align: center;
		text-decoration: none;
		border: 1px solid #eeeeee;
	}

	.feature-card:hover {
		transform: translateY(-5px);
		box-shadow: 0 10px 25px rgba(0, 0, 0, 0.15);
	}

	.feature-card .card-bg {
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		z-index: 1;
	}

	.feature-card .card-bg img,
	.feature-card .card-bg .gradient-bg {
		width: 100%;
		height: 100%;
		object-fit: cover;
		transition: transform 0.5s ease;
	}

	.feature-card .card-bg::after {
		content: "";
		position: absolute;
		top: 0;
		left: 0;
		width: 100%;
		height: 100%;
		background: rgba(0, 0, 0, 0.35);
		transition: background 0.3s ease;
		z-index: 2;
	}

	.feature-card:hover .card-bg::after {
		background: rgba(0, 0, 0, 0.25);
	}

	.feature-card:hover .card-bg img {
		transform: scale(1.1);
	}

	.feature-card .card-content {
		position: relative;
		z-index: 3;
		padding: 1.5rem;
	}

	.feature-card h2 {
		margin: 0;
		font-size: 2rem;
		font-weight: 800;
		color: #ffffff;
		text-shadow: 0 4px 12px rgba(0, 0, 0, 0.6);
		letter-spacing: -0.02em;
	}

	.gradient-bg {
		width: 100%;
		height: 100%;
	}

	.gallery-gradient {
		background: linear-gradient(135deg, #4facfe, #00f2fe);
	}

	.shop-gradient {
		background: linear-gradient(135deg, #667eea, #764ba2);
	}

	.games-gradient {
		background: linear-gradient(135deg, #ff416c, #ff4b2b);
	}

	@media (max-width: 768px) {
		.carousel-container {
			height: 320px;
		}
		.carousel-track {
			height: 260px;
		}
		.carousel-slide {
			width: 85%;
		}
		.carousel-slide.left {
			transform: translate(-105%, -50%) scale(0.8);
		}
		.carousel-slide.right {
			transform: translate(5%, -50%) scale(0.8);
		}
	}
</style>
