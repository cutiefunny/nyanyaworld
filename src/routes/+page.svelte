<script>
	import { db } from "$lib/firebase";
	import { collection, onSnapshot, query, limit } from "firebase/firestore";

	let records = $state([]);

	$effect(() => {
		const q = query(collection(db, "records"), limit(10));
		const unsubscribe = onSnapshot(q, (snapshot) => {
			records = snapshot.docs.map((doc) => ({
				id: doc.id,
				...doc.data(),
			}));
		});
		return unsubscribe;
	});
</script>

<div class="container">
	<section class="hero">
		<h1>냐냐월드</h1>
		<p class="subtitle">모든 것이 냐냐로 통하는 세상</p>
		<div class="cta-buttons">
			<a href="/about" class="btn btn-primary">시작하기</a>
		</div>
	</section>

	<section
		class="records-section"
		style="margin-top: 3rem; text-align: center;"
	>
		<h2>최신 레코드</h2>
		{#if records.length === 0}
			<p style="color: #666;">데이터가 존재하지 않습니다.</p>
		{:else}
			<div
				style="display: grid; gap: 1rem; max-width: 600px; margin: 1.5rem auto;"
			>
				{#each records as record}
					<div
						style="padding: 1rem; background: #f9f9f9; border: 1px solid #eee; border-radius: 8px; text-align: left;"
					>
						<pre
							style="margin: 0; font-size: 0.8rem; white-space: pre-wrap;">{JSON.stringify(
								record,
								null,
								2,
							)}</pre>
					</div>
				{/each}
			</div>
		{/if}
	</section>

	<section class="features">
		<div class="feature-card">
			<h2>만화보기</h2>
			<p>냐냐월드의 재미있는 만화들을 감상하세요.</p>
			<a
				href="/toons"
				class="btn btn-secondary"
				style="margin-top: 1rem; display: inline-block;">보러가기</a
			>
		</div>
		<div class="feature-card">
			<h2>게임하기</h2>
			<p>다양한 미니 게임들을 즐길 수 있습니다.</p>
			<a
				href="/games"
				class="btn btn-secondary"
				style="margin-top: 1rem; display: inline-block;">플레이하기</a
			>
		</div>
		<div class="feature-card">
			<h2>쇼핑하기</h2>
			<p>귀여운 굿즈와 아이템을 만나보세요.</p>
			<a
				href="/shop"
				class="btn btn-secondary"
				style="margin-top: 1rem; display: inline-block;"
				>쇼핑하러 가기</a
			>
		</div>
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

	.subtitle {
		font-size: 1.25rem;
		color: #666666;
		margin-bottom: 2rem;
	}

	.cta-buttons {
		display: flex;
		gap: 1rem;
		justify-content: center;
		flex-wrap: wrap;
	}

	.btn {
		padding: 0.75rem 2rem;
		font-size: 1rem;
		border: none;
		border-radius: 4px;
		cursor: pointer;
		transition: all 0.3s ease;
		font-weight: 600;
	}

	.btn-primary {
		background-color: #0066cc;
		color: #ffffff;
	}

	.btn-primary:hover {
		background-color: #0052a3;
	}

	.btn-secondary {
		background-color: #f0f0f0;
		color: #000000;
		border: 1px solid #dddddd;
	}

	.btn-secondary:hover {
		background-color: #e6e6e6;
	}

	.features {
		display: grid;
		grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
		gap: 2rem;
		padding: 3rem 0;
	}

	.feature-card {
		padding: 2rem;
		border: 1px solid #eeeeee;
		border-radius: 8px;
		background-color: #fafafa;
		transition: all 0.3s ease;
	}

	.feature-card:hover {
		border-color: #dddddd;
		box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
	}

	.feature-card h2 {
		font-size: 1.5rem;
		color: #000000;
		margin-bottom: 1rem;
	}

	.feature-card p {
		color: #666666;
		line-height: 1.8;
	}

	@media (max-width: 768px) {
		.hero h1 {
			font-size: 2rem;
		}

		.subtitle {
			font-size: 1rem;
		}

		.cta-buttons {
			flex-direction: column;
			align-items: center;
		}

		.btn {
			width: 100%;
			max-width: 300px;
		}
	}
</style>
