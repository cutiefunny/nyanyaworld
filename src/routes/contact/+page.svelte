<div class="container">
    <section class="page-header">
        <h1>연락처</h1>
        <p class="subtitle">궁금한 점이 있으신가요? 언제든지 연락주세요.</p>
    </section>

    <div class="contact-wrapper">
        <div class="contact-info">
            <div class="info-item">
                <h3>이메일</h3>
                <p>contact@nyanyaworld.com</p>
            </div>
            <div class="info-item">
                <h3>SNS</h3>
                <p>@nyanyaworld_official</p>
            </div>
            <div class="info-item">
                <h3>주소</h3>
                <p>고양이 행성 은하수길 22</p>
            </div>
        </div>

        <form class="contact-form" onsubmit={(e) => e.preventDefault()}>
            <div class="form-group">
                <label for="name">이름</label>
                <input type="text" id="name" placeholder="이름을 입력하세요" />
            </div>
            <div class="form-group">
                <label for="email">이메일</label>
                <input
                    type="email"
                    id="email"
                    placeholder="email@example.com"
                />
            </div>
            <div class="form-group">
                <label for="message">메시지</label>
                <textarea
                    id="message"
                    rows="5"
                    placeholder="궁금한 내용을 적어주세요"
                ></textarea>
            </div>
            <button class="btn btn-primary">보내기</button>
        </form>
    </div>

    <div class="actions">
        <a href="/" class="btn btn-secondary">홈으로 돌아가기</a>
    </div>
</div>

<style>
    .container {
        max-width: 900px;
        margin: 0 auto;
    }

    .page-header {
        text-align: center;
        margin-bottom: 3rem;
    }

    .page-header h1 {
        font-size: 2.5rem;
        color: #000;
    }

    .contact-wrapper {
        display: grid;
        grid-template-columns: 1fr 2fr;
        gap: 3rem;
        background: #fff;
        padding: 2rem;
        border-radius: 12px;
        border: 1px solid #eee;
        margin-bottom: 3rem;
    }

    .contact-info {
        display: flex;
        flex-direction: column;
        gap: 2rem;
    }

    .info-item h3 {
        font-size: 1.1rem;
        color: #0066cc;
        margin-bottom: 0.5rem;
    }

    .info-item p {
        color: #444;
    }

    .contact-form {
        display: flex;
        flex-direction: column;
        gap: 1.5rem;
    }

    .form-group {
        display: flex;
        flex-direction: column;
        gap: 0.5rem;
    }

    .form-group label {
        font-weight: 600;
        font-size: 0.9rem;
    }

    .form-group input,
    .form-group textarea {
        padding: 0.75rem;
        border: 1px solid #ddd;
        border-radius: 6px;
        font-size: 1rem;
    }

    .btn {
        display: inline-block;
        padding: 0.75rem 2rem;
        font-size: 1rem;
        border-radius: 4px;
        font-weight: 600;
        text-decoration: none;
        border: none;
        cursor: pointer;
        transition: all 0.3s ease;
    }

    .btn-primary {
        background-color: #0066cc;
        color: #fff;
    }

    .btn-primary:hover {
        background-color: #0052a3;
    }

    .btn-secondary {
        background-color: #f0f0f0;
        color: #000;
        border: 1px solid #ddd;
    }

    .actions {
        text-align: center;
    }

    @media (max-width: 768px) {
        .contact-wrapper {
            grid-template-columns: 1fr;
        }
    }
</style>
