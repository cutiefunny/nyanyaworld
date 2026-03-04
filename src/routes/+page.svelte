<script>
	import { db } from "$lib/firebase";
	import { collection, onSnapshot, query, limit } from "firebase/firestore";

	let records = $state([]);
	let randomComicThumbnail = $state(null);
	let randomGameThumbnail = $state(null);

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

		return () => {
			unsubscribeRecords();
			unsubscribeComics();
			unsubscribeGames();
		};
	});
</script>

<div class="container">
	<section class="hero">
		<h1>(최신 소식 슬라이드)</h1>
	</section>

	<section class="features">
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

		<a href="/shop" class="feature-card shop-card">
			<div class="card-bg">
				<!-- 쇼핑 섹션용 배경 (이미지 부재 시 그라데이션) -->
				<div class="gradient-bg shop-gradient"></div>
			</div>
			<div class="card-content">
				<h2>쇼핑하기</h2>
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
		padding: 3rem 0;
	}

	.hero h1 {
		font-size: 3rem;
		color: #000000;
		margin-bottom: 1rem;
		font-weight: 700;
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

	.games-gradient {
		background: linear-gradient(135deg, #ff416c, #ff4b2b);
	}

	.shop-gradient {
		background: linear-gradient(135deg, #4facfe, #00f2fe);
	}

	@media (max-width: 768px) {
		.hero h1 {
			font-size: 2rem;
		}
	}
</style>
