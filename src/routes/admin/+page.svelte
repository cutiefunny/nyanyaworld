<script>
    import { db } from "$lib/firebase";
    import {
        collection,
        onSnapshot,
        query,
        orderBy,
        getDocs,
        limit,
    } from "firebase/firestore";

    // Stats
    let totalStats = $state(0);
    let recentStats = $state([]);
    let statsLoading = $state(true);

    // Pagination
    let currentPage = $state(1);
    const pageSize = 10;
    let paginatedStats = $derived(
        recentStats.slice((currentPage - 1) * pageSize, currentPage * pageSize),
    );
    let totalPages = $derived(Math.ceil(recentStats.length / pageSize));

    $effect(() => {
        // Fetch Stats
        const statsRef = collection(db, "Stats");
        const qStats = query(
            statsRef,
            orderBy("timestamp", "desc"),
            limit(100),
        );
        const unsubStats = onSnapshot(qStats, (snapshot) => {
            recentStats = snapshot.docs.map((doc) => ({
                id: doc.id,
                ...doc.data(),
            }));
        });

        // Total count
        getDocs(statsRef).then((snap) => {
            totalStats = snap.size;
            statsLoading = false;
        });

        return () => {
            unsubStats();
        };
    });
</script>

<section class="admin-section">
    <header class="section-header">
        <h2>대시보드 / 접속 통계</h2>
    </header>

    <div class="stats-overview">
        <div class="stat-box">
            <span class="label">총 누적 방문자</span>
            <span class="value">{statsLoading ? "..." : totalStats}</span>
        </div>
    </div>

    <div class="recent-stats stats-card">
        <div class="card-header">
            <h3>최근 실시간 로그 (Top 100)</h3>
            <div class="pagination-controls">
                <button
                    disabled={currentPage === 1}
                    onclick={() => (currentPage -= 1)}>이전</button
                >
                <span class="page-info">{currentPage} / {totalPages || 1}</span>
                <button
                    disabled={currentPage >= totalPages}
                    onclick={() => (currentPage += 1)}>다음</button
                >
            </div>
        </div>
        <div class="scroll-area">
            <table>
                <thead>
                    <tr>
                        <th>경로</th>
                        <th>기기 정보</th>
                        <th>시간</th>
                    </tr>
                </thead>
                <tbody>
                    {#each paginatedStats as stat}
                        <tr>
                            <td><code>{stat.path}</code></td>
                            <td class="ua"
                                >{stat.userAgent?.substring(0, 30)}...</td
                            >
                            <td class="time"
                                >{stat.timestamp?.toDate().toLocaleString() ||
                                    "..."}</td
                            >
                        </tr>
                    {/each}
                </tbody>
            </table>
        </div>
    </div>
</section>

<style>
    .admin-section {
        max-width: 1200px;
        margin: 0 auto;
    }

    .section-header {
        margin-bottom: 2rem;
    }

    .section-header h2 {
        font-size: 1.75rem;
        color: #333;
        margin: 0;
    }

    /* Stats */
    .stats-overview {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
        gap: 1.5rem;
        margin-bottom: 2rem;
    }

    .stat-box {
        background: #fff;
        padding: 2rem;
        border-radius: 16px;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
        display: flex;
        flex-direction: column;
    }

    .stat-box .label {
        color: #666;
        font-size: 0.9rem;
        margin-bottom: 0.5rem;
    }

    .stat-box .value {
        font-size: 2.5rem;
        font-weight: 800;
        color: #0066cc;
    }

    .scroll-area {
        background: #fff;
        border-radius: 12px;
        box-shadow: 0 4px 20px rgba(0, 0, 0, 0.05);
        overflow: hidden;
    }

    table {
        width: 100%;
        border-collapse: collapse;
    }

    th,
    td {
        padding: 0.5rem 1.5rem;
        text-align: left;
        border-bottom: 1px solid #f0f0f0;
    }

    th {
        background: #f8f9fa;
        font-size: 0.85rem;
        color: #666;
        text-transform: uppercase;
        letter-spacing: 0.05em;
    }

    td {
        font-size: 0.9rem;
    }

    td code {
        background: #f1f3f5;
        padding: 0.2rem 0.4rem;
        border-radius: 4px;
        font-size: 0.8rem;
    }

    .ua {
        color: #999;
        font-size: 0.8rem;
    }

    .time {
        white-space: nowrap;
        font-family: monospace;
        font-size: 0.85rem;
    }

    .card-header {
        display: flex;
        justify-content: space-between;
        align-items: center;
        padding: 0 1rem;
        margin-bottom: 1rem;
    }

    .pagination-controls {
        display: flex;
        align-items: center;
        gap: 1rem;
    }

    .pagination-controls button {
        padding: 0.4rem 0.8rem;
        border: 1px solid #ddd;
        border-radius: 4px;
        background: #fff;
        cursor: pointer;
        font-size: 0.85rem;
    }

    .pagination-controls button:disabled {
        opacity: 0.5;
        cursor: not-allowed;
    }

    .page-info {
        font-size: 0.9rem;
        color: #666;
        min-width: 60px;
        text-align: center;
    }
</style>
