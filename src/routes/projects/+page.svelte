<div class="container">
    <section class="page-header">
        <h1>프로젝트</h1>
        <p class="subtitle">
            냐냐월드에서 진행 중인 멋진 프로젝트들을 소개합니다.
        </p>
    </section>

    <section class="projects-grid">
        <div class="project-card">
            <div class="project-img-placeholder">Project 1</div>
            <div class="project-info">
                <h2>스마트 캣 피더</h2>
                <p>IoT 기반의 자동 고양이 급식기 프로젝트입니다.</p>
                <span class="tag">Hardware</span>
            </div>
        </div>

        <div class="project-card">
            <div class="project-img-placeholder">Project 2</div>
            <div class="project-info">
                <h2>야옹이 사전</h2>
                <p>고양이의 언어와 행동을 분석해주는 AI 앱입니다.</p>
                <span class="tag">AI / Mobile</span>
            </div>
        </div>
    </section>

    <div class="actions">
        <a href="/" class="btn btn-secondary">홈으로 돌아가기</a>
    </div>
</div>

<style>
    .container {
        max-width: 1000px;
        margin: 0 auto;
    }

    .page-header {
        text-align: center;
        margin-bottom: 3rem;
    }

    .page-header h1 {
        font-size: 2.5rem;
        margin-bottom: 1rem;
        color: #000;
    }

    .subtitle {
        color: #666;
    }

    .projects-grid {
        display: grid;
        grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
        gap: 2rem;
        margin-bottom: 3rem;
    }

    .project-card {
        background: #fff;
        border: 1px solid #eee;
        border-radius: 12px;
        overflow: hidden;
        transition:
            transform 0.3s ease,
            box-shadow 0.3s ease;
    }

    .project-card:hover {
        transform: translateY(-5px);
        box-shadow: 0 10px 20px rgba(0, 0, 0, 0.05);
    }

    .project-img-placeholder {
        height: 200px;
        background: #f0f0f0;
        display: flex;
        align-items: center;
        justify-content: center;
        font-weight: bold;
        color: #999;
    }

    .project-info {
        padding: 1.5rem;
    }

    .project-info h2 {
        font-size: 1.25rem;
        margin-bottom: 0.5rem;
        color: #333;
    }

    .project-info p {
        color: #666;
        margin-bottom: 1rem;
        font-size: 0.95rem;
    }

    .tag {
        background: #eef6ff;
        color: #0066cc;
        padding: 0.25rem 0.75rem;
        border-radius: 20px;
        font-size: 0.8rem;
        font-weight: 600;
    }

    .actions {
        text-align: center;
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
</style>
