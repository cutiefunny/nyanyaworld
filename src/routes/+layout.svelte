<script>
	import favicon from "$lib/assets/favicon.svg";
	import { page } from "$app/state";
	import { db } from "$lib/firebase";
	import { collection, addDoc, serverTimestamp } from "firebase/firestore";

	let { children } = $props();

	$effect(() => {
		const path = page.url.pathname;
		// 관리자 페이지 접속은 카운트에서 제외하거나 별도 처리 가능
		if (path.startsWith("/admin")) return;

		const logVisit = async () => {
			try {
				await addDoc(collection(db, "Stats"), {
					path,
					userAgent: navigator.userAgent,
					timestamp: serverTimestamp(),
				});
			} catch (e) {
				console.error("Stats logging failed:", e);
			}
		};
		logVisit();
	});
</script>

<svelte:head>
	<link rel="icon" href={favicon} />
	<style global>
		* {
			margin: 0;
			padding: 0;
			box-sizing: border-box;
		}

		body {
			font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto,
				Oxygen, Ubuntu, Cantarell, sans-serif;
			background-color: #ffffff;
			color: #333333;
			line-height: 1.6;
		}

		a {
			color: #0066cc;
			text-decoration: none;
			transition: color 0.2s ease;
		}

		a:hover {
			color: #0052a3;
		}
	</style>
</svelte:head>

<header>
	<nav class="navbar">
		<div class="container">
			<h1 class="logo">냐냐월드</h1>
			<ul class="nav-links">
				<li><a href="/">Home</a></li>
				<li><a href="/toons">Toons</a></li>
				<li><a href="/games">Games</a></li>
				<li><a href="/shop">Shop</a></li>
			</ul>
		</div>
	</nav>
</header>

<main>
	{@render children()}
</main>

<footer>
	<div class="container">
		<p>&copy; 2026 냐냐월드. All rights reserved.</p>
	</div>
</footer>

<style>
	header {
		display: none;
		border-bottom: 1px solid #eeeeee;
	}

	@media (min-width: 1024px) {
		header {
			display: block;
		}
	}

	.navbar {
		padding: 1rem 0;
	}

	.container {
		max-width: 1200px;
		margin: 0 auto;
		padding: 0 2rem;
		display: flex;
		justify-content: space-between;
		align-items: center;
	}

	.logo {
		font-size: 1.5rem;
		font-weight: 700;
		color: #000000;
	}

	.nav-links {
		display: flex;
		list-style: none;
		gap: 2rem;
	}

	main {
		min-height: calc(100vh - 200px);
		padding: 1rem 0.5rem;
	}

	footer {
		border-top: 1px solid #eeeeee;
		padding: 2rem 0;
		text-align: center;
		color: #666666;
		font-size: 0.875rem;
	}
</style>
