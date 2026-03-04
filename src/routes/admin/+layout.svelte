<script>
    import { page } from "$app/state";

    let { children } = $props();

    let currentPath = $derived(page.url.pathname);
</script>

<div class="admin-wrapper">
    <!-- Sidebar -->
    <aside class="sidebar">
        <div class="sidebar-header">
            <h2>ADMIN</h2>
        </div>
        <nav class="sidebar-nav">
            <a href="/admin" class:active={currentPath === "/admin"}>
                <span class="icon">📊</span> 접속 통계
            </a>
            <a
                href="/admin/games"
                class:active={currentPath === "/admin/games"}
            >
                <span class="icon">🎮</span> 게임 컨텐츠 관리
            </a>
            <a
                href="/admin/gallery"
                class:active={currentPath === "/admin/gallery"}
            >
                <span class="icon">🖼️</span> 갤러리 관리
            </a>
        </nav>
        <div class="sidebar-footer">
            <a href="/" class="home-link">홈으로 돌아가기</a>
        </div>
    </aside>

    <!-- Main Content Area -->
    <main class="content-area">
        {@render children()}
    </main>
</div>

<style>
    :global(body) {
        background-color: #f0f2f5;
        overflow: hidden;
    }

    .admin-wrapper {
        display: flex;
        height: 100vh;
        width: 100vw;
    }

    /* Sidebar */
    .sidebar {
        width: 260px;
        background-color: #1a1c23;
        color: #fff;
        display: flex;
        flex-direction: column;
        flex-shrink: 0;
    }

    .sidebar-header {
        padding: 2rem 1.5rem;
        border-bottom: 1px solid #2d2f39;
    }

    .sidebar-header h2 {
        font-size: 1.25rem;
        font-weight: 800;
        letter-spacing: 0.1em;
        margin: 0;
        color: #0066cc;
    }

    .sidebar-nav {
        flex-grow: 1;
        padding: 1.5rem 0;
    }

    .sidebar-nav a {
        width: 100%;
        padding: 1rem 1.5rem;
        background: none;
        border: none;
        color: #adb5bd;
        text-align: left;
        cursor: pointer;
        font-size: 0.95rem;
        font-weight: 600;
        display: flex;
        align-items: center;
        gap: 0.75rem;
        transition: all 0.2s;
        text-decoration: none;
    }

    .sidebar-nav a:hover {
        background: #24262d;
        color: #fff;
    }

    .sidebar-nav a.active {
        background: #0066cc;
        color: #fff;
    }

    .sidebar-footer {
        padding: 1.5rem;
        border-top: 1px solid #2d2f39;
    }

    .home-link {
        font-size: 0.85rem;
        color: #666;
        text-decoration: none;
    }

    .home-link:hover {
        color: #fff;
    }

    /* Content Area */
    .content-area {
        flex-grow: 1;
        overflow-y: auto;
        padding: 2.5rem;
    }
</style>
