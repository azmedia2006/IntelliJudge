# IntelliJudge

IntelliJudge là hệ thống chấm bài lập trình trực tuyến dành cho học sinh, sinh viên và giảng viên. Hệ thống cho phép quản lý đề bài, test case, tài khoản, nhận bài nộp, chạy mã nguồn trong môi trường Judge0, cập nhật bảng xếp hạng và sử dụng mô hình ngôn ngữ để tạo gợi ý khi bài làm chưa chính xác.

> Trạng thái hiện tại: dự án phục vụ mục đích học tập/demo. Một số màn hình và dữ liệu thống kê ở frontend vẫn đang ở dạng mô phỏng; cấu hình bảo mật cần được siết chặt trước khi triển khai production.

## Mục lục

- [Tính năng](#tính-năng)
- [Kiến trúc hệ thống](#kiến-trúc-hệ-thống)
- [Luồng chấm bài](#luồng-chấm-bài)
- [Công nghệ sử dụng](#công-nghệ-sử-dụng)
- [Cấu trúc thư mục](#cấu-trúc-thư-mục)
- [Mô hình dữ liệu](#mô-hình-dữ-liệu)
- [Cài đặt và chạy dự án](#cài-đặt-và-chạy-dự-án)
- [Cấu hình môi trường](#cấu-hình-môi-trường)
- [Khởi tạo dữ liệu](#khởi-tạo-dữ-liệu)
- [Import đề bài bằng ZIP](#import-đề-bài-bằng-zip)
- [API chính](#api-chính)
- [Phân quyền](#phân-quyền)
- [Trạng thái bài nộp](#trạng-thái-bài-nộp)
- [Các điểm cần lưu ý](#các-điểm-cần-lưu-ý)
- [Hướng phát triển](#hướng-phát-triển)

## Tính năng

### Người dùng

- Đăng ký tài khoản học sinh.
- Đăng nhập bằng username hoặc email.
- Xem danh sách và chi tiết bài tập.
- Lọc bài tập theo tag và tìm kiếm theo tên/ID.
- Viết code bằng Monaco Editor.
- Nộp bài bằng C++, Java hoặc Python.
- Xem lịch sử và chi tiết bài nộp.
- Nhận trạng thái chấm, thời gian chạy, bộ nhớ sử dụng và gợi ý từ AI.
- Xem bảng xếp hạng.

### Quản trị viên

- Xem, tạo, sửa và xóa tài khoản theo phạm vi quyền hạn.
- Tạo, sửa và xóa đề bài.
- Quản lý test case công khai và test case ẩn.
- Import đề bài cùng test case từ file ZIP.
- Xem toàn bộ bài nộp của hệ thống.
- Quản lý bài viết/hướng dẫn thông qua API.

### Hệ thống chấm

- Đẩy yêu cầu chấm sang Celery để API không phải chờ quá trình chạy code.
- Sử dụng Redis làm message broker và result backend.
- Gọi Judge0 để biên dịch/chạy code trong sandbox.
- Chạy tuần tự qua các test case và dừng tại test case lỗi đầu tiên.
- Ánh xạ kết quả về các trạng thái AC, WA, TLE, MLE, CE hoặc SYSTEM_ERROR.
- Gọi LLM tương thích OpenAI API để tạo gợi ý cho bài sai mà không chủ động cung cấp lời giải hoàn chỉnh.
- Tính lại bảng xếp hạng sau khi chấm xong.

## Kiến trúc hệ thống

```mermaid
flowchart LR
    U[Trình duyệt] -->|HTTP/JSON + JWT| FE[React + Vite]
    FE -->|REST API| API[FastAPI]
    API --> DB[(MySQL)]
    API -->|Tạo task| R[(Redis)]
    R --> CW[Celery Worker]
    CW -->|Đọc đề và test case| DB
    CW -->|Chạy mã nguồn| J[Judge0]
    CW -->|Sinh gợi ý khi lỗi| LLM[LLM OpenAI-compatible]
    CW -->|Cập nhật kết quả và ranking| DB
    FE -->|Lấy trạng thái bài nộp| API
```

### Vai trò của từng thành phần

| Thành phần | Trách nhiệm |
|---|---|
| React/Vite | Giao diện người dùng, router, form đăng nhập/đăng ký, Monaco Editor và trang quản trị |
| FastAPI | Xác thực, phân quyền, CRUD dữ liệu và cung cấp REST API |
| SQLAlchemy Async | Làm việc bất đồng bộ với MySQL |
| MySQL | Lưu user, đề bài, test case, bài nộp, bài viết và ranking |
| Redis | Hàng đợi cho Celery và nơi lưu kết quả task |
| Celery | Xử lý chấm bài ở tiến trình nền |
| Judge0 | Biên dịch và chạy code trong sandbox |
| LLM | Phân tích lỗi và tạo gợi ý bằng tiếng Việt |

## Luồng chấm bài

1. Người dùng đăng nhập và nhận JWT.
2. Frontend gửi `problem_id`, `language` và `code` tới `POST /api/v1/submissions`.
3. Backend kiểm tra đề bài, tạo bản ghi `Submission` với trạng thái `PENDING`.
4. Backend gọi `process_submission_task.delay(submission_id)` để đẩy task sang Redis.
5. Celery Worker tải bài nộp, đề bài và toàn bộ test case từ MySQL.
6. Worker ánh xạ ngôn ngữ sang Judge0:
   - `cpp` → language ID `54`
   - `java` → language ID `62`
   - `python` → language ID `71`
7. Mỗi test case được gửi tới Judge0 với giới hạn thời gian và bộ nhớ của đề bài.
8. Nếu một test case không đạt, worker dừng ngay để tiết kiệm tài nguyên.
9. Với CE, hệ thống lưu thông báo biên dịch. Với WA/TLE/MLE, hệ thống gọi LLM để sinh gợi ý nếu có đủ dữ liệu lỗi.
10. Kết quả, thời gian lớn nhất và bộ nhớ lớn nhất được ghi lại vào MySQL.
11. Ranking của người dùng được tính lại.
12. Frontend truy vấn danh sách/chi tiết bài nộp để hiển thị kết quả.

## Công nghệ sử dụng

### Frontend

- React 19
- React Router
- Vite
- Tailwind CSS
- Axios
- Monaco Editor
- Lucide React
- React Markdown

### Backend

- Python 3.10+
- FastAPI
- Uvicorn
- SQLAlchemy 2 Async
- aiomysql
- Pydantic Settings
- JWT + bcrypt/passlib
- Celery
- Redis
- HTTPX

### Hạ tầng tích hợp

- MySQL 8
- Redis
- Judge0
- Ollama hoặc một API tương thích OpenAI Chat Completions

## Cấu trúc thư mục

```text
IntelliJudge-main/
├── backend/
│   ├── app/
│   │   ├── api/
│   │   │   ├── dependencies.py       # Đọc JWT và kiểm tra quyền
│   │   │   └── v1/
│   │   │       ├── router.py         # Đăng ký toàn bộ API v1
│   │   │       └── endpoints/        # Auth, users, problems, submissions...
│   │   ├── core/
│   │   │   ├── config.py             # Đọc biến môi trường
│   │   │   ├── database.py           # Async engine và DB session
│   │   │   └── security.py           # Hash mật khẩu và tạo JWT
│   │   ├── models/                    # SQLAlchemy models
│   │   ├── schemas/                   # Pydantic request/response schemas
│   │   ├── services/
│   │   │   ├── sandbox.py            # Client gọi Judge0
│   │   │   └── ai_agent.py           # Client gọi LLM
│   │   ├── worker/
│   │   │   ├── celery_app.py         # Cấu hình Celery
│   │   │   └── tasks.py              # Logic chấm và tính ranking
│   │   └── main.py                    # FastAPI application
│   ├── scratch/
│   │   ├── create_tables.py           # Tạo bảng database
│   │   ├── create_admin.py            # Tạo admin mặc định
│   │   ├── seed_api.py                # Seed đề bài qua API
│   │   └── add_tags_column.py         # Script migration thủ công cũ
│   ├── .env.example
│   ├── requirements.txt
│   └── run.py                         # Chạy Uvicorn và Celery cùng lúc
├── frontend/
│   ├── src/
│   │   ├── components/                # Navbar, bảng bài tập, sidebar
│   │   ├── contexts/AuthContext.jsx   # Quản lý token và user hiện tại
│   │   ├── layouts/MainLayout.jsx
│   │   ├── pages/                     # Các trang người dùng
│   │   ├── pages/admin/               # Quản lý user và đề bài
│   │   ├── services/api.js            # Axios instance + JWT interceptor
│   │   ├── App.jsx                    # Router và route guard
│   │   └── main.jsx
│   ├── .env.example
│   └── package.json
├── docker-compose.yml                 # MySQL và Redis
├── problems.json                      # Dữ liệu đề bài mẫu
└── README.md
```

## Mô hình dữ liệu

```mermaid
erDiagram
    USER ||--o{ SUBMISSION : submits
    USER ||--o{ PROBLEM : creates
    USER ||--o{ ARTICLE : writes
    USER ||--o| RANKING : has
    PROBLEM ||--o{ TEST_CASE : contains
    PROBLEM ||--o{ SUBMISSION : receives

    USER {
        int id PK
        string username UK
        string email UK
        string hashed_password
        enum role
        datetime created_at
        datetime updated_at
    }

    PROBLEM {
        int id PK
        string title
        text description
        float time_limit
        float memory_limit
        string tags
        int created_by_id FK
    }

    TEST_CASE {
        int id PK
        int problem_id FK
        text input_data
        text output_data
        bool is_hidden
    }

    SUBMISSION {
        int id PK
        int problem_id FK
        int user_id FK
        text code
        string language
        enum status
        float execution_time
        float memory_used
        text ai_hint
    }

    RANKING {
        int id PK
        int user_id FK
        int solved_count
        float total_time
        int penalty
    }
```

### Cách tính ranking hiện tại

Ranking được sắp xếp theo:

1. `solved_count` giảm dần.
2. `total_time` tăng dần.
3. `penalty` tăng dần.

Cách tính trong mã nguồn là kiểu **ICPC-inspired**, chưa phải cách tính ICPC chuẩn:

- `solved_count`: số đề khác nhau có ít nhất một bài AC.
- `total_time`: tổng thời gian thực thi của lần AC đầu tiên cho từng đề.
- `penalty`: số lần WA/TLE/MLE trước lần AC đầu tiên; CE không được tính.

## Cài đặt và chạy dự án

### 1. Yêu cầu

- Git
- Python 3.10 trở lên
- Node.js 18 trở lên
- Docker và Docker Compose
- Một Judge0 instance có thể truy cập qua HTTP
- Tùy chọn: Ollama hoặc API LLM tương thích OpenAI

### 2. Khởi động MySQL và Redis

Tại thư mục gốc:

```bash
docker compose up -d
```

Kiểm tra container:

```bash
docker compose ps
```

Docker Compose hiện tạo:

- MySQL tại `localhost:3306`
- Redis tại `localhost:6379`
- Database mặc định: `intellijudge_db`

### 3. Cài đặt backend

```bash
cd backend
python -m venv venv
```

Kích hoạt môi trường ảo:

**Windows PowerShell**

```powershell
.\venv\Scripts\Activate.ps1
```

**Windows CMD**

```bat
venv\Scripts\activate.bat
```

**Linux/macOS**

```bash
source venv/bin/activate
```

Cài thư viện:

```bash
pip install -r requirements.txt
```

Tạo file cấu hình:

**Windows**

```bat
copy .env.example .env
```

**Linux/macOS**

```bash
cp .env.example .env
```

Mở `.env` và sửa `DATABASE_URL` thành tên database đúng với Docker Compose:

```env
DATABASE_URL="mysql+aiomysql://root:rootpassword@localhost:3306/intellijudge_db"
```

Khởi tạo bảng:

```bash
python scratch/create_tables.py
```

Tạo tài khoản admin mẫu:

```bash
python scratch/create_admin.py
```

Tài khoản được script tạo sẵn:

```text
Username: admin
Password: admin123
```

> Hãy đổi tài khoản/mật khẩu mặc định trước khi triển khai thật.

### 4. Chạy backend và Celery

Cách nhanh nhất:

```bash
python run.py
```

Script này mở đồng thời:

- FastAPI/Uvicorn tại `http://localhost:8000`
- Celery Worker

Có thể chạy riêng trong hai terminal để dễ theo dõi log:

**Terminal 1 — API**

```bash
cd backend
python -m uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

**Terminal 2 — Worker**

```bash
cd backend
python -m celery -A app.worker.tasks.celery_app worker --loglevel=info --pool=solo
```

Tài liệu API tự động:

- Swagger UI: `http://localhost:8000/docs`
- ReDoc: `http://localhost:8000/redoc`
- Health check: `http://localhost:8000/`

### 5. Cài đặt frontend

Mở terminal mới:

```bash
cd frontend
npm install
```

Tạo file môi trường:

**Windows**

```bat
copy .env.example .env
```

**Linux/macOS**

```bash
cp .env.example .env
```

Nội dung mặc định:

```env
VITE_API_URL=http://localhost:8000/api/v1
```

Chạy frontend:

```bash
npm run dev
```

Mở địa chỉ Vite hiển thị trong terminal, thông thường là:

```text
http://localhost:5173
```

### 6. Cấu hình Judge0

Judge0 không được khai báo trong `docker-compose.yml` của dự án này. Bạn phải chạy Judge0 riêng hoặc sử dụng một Judge0 API có sẵn, sau đó cấu hình:

```env
JUDGE0_API_URL="http://localhost:2358"
JUDGE0_API_KEY=""
```

Backend gọi endpoint:

```text
POST {JUDGE0_API_URL}/submissions?base64_encoded=true&wait=true
```

Nếu Judge0 không hoạt động, hệ thống không thể chấm code chính xác.

### 7. Cấu hình LLM tùy chọn

Mặc định dự án kỳ vọng một API tương thích OpenAI tại:

```env
LLM_API_URL="http://localhost:11434/v1"
LLM_API_KEY="ollama"
LLM_MODEL="qwen2.5-coder:7b"
```

Ví dụ trên phù hợp với Ollama có hỗ trợ endpoint tương thích OpenAI. Nếu không cấu hình LLM, việc chấm vẫn có thể chạy nhưng gợi ý AI sẽ trả về thông báo dự phòng khi kết nối thất bại.

## Cấu hình môi trường

### Backend `.env`

| Biến | Ý nghĩa | Ví dụ |
|---|---|---|
| `PROJECT_NAME` | Tên hiển thị trong FastAPI | `IntelliJudge` |
| `API_V1_STR` | Prefix API | `/api/v1` |
| `DEBUG` | Bật SQL log và chế độ debug | `true` |
| `DATABASE_URL` | Chuỗi kết nối MySQL async | `mysql+aiomysql://.../intellijudge_db` |
| `JWT_SECRET_KEY` | Khóa ký JWT | Chuỗi bí mật dài, ngẫu nhiên |
| `JWT_ALGORITHM` | Thuật toán JWT | `HS256` |
| `ACCESS_TOKEN_EXPIRE_MINUTES` | Thời hạn token | `1440` |
| `REDIS_URL` | Celery broker/backend | `redis://localhost:6379/0` |
| `JUDGE0_API_URL` | URL Judge0 | `http://localhost:2358` |
| `JUDGE0_API_KEY` | API key nếu dịch vụ yêu cầu | Để trống khi chạy local |
| `LLM_API_URL` | Base URL tương thích OpenAI | `http://localhost:11434/v1` |
| `LLM_API_KEY` | Khóa LLM | `ollama` hoặc API key thật |
| `LLM_MODEL` | Tên model | `qwen2.5-coder:7b` |

### Frontend `.env`

| Biến | Ý nghĩa |
|---|---|
| `VITE_API_URL` | Base URL của FastAPI API v1 |

## Khởi tạo dữ liệu

File `problems.json` chứa dữ liệu bài tập mẫu. Script seed gửi dữ liệu qua API nên backend phải chạy và tài khoản admin phải tồn tại.

Script `seed_api.py` sử dụng thư viện `requests`, nhưng thư viện này hiện chưa được khai báo trong `requirements.txt`. Cài bổ sung trước khi chạy:

```bash
pip install requests
python scratch/seed_api.py
```

Script thử đăng nhập lần lượt bằng:

1. `admin_root / IntelliJudge@123`
2. `admin / admin123`

Sau đó script tạo đề bài và test case bằng REST API.

## Import đề bài bằng ZIP

Admin có thể import một đề bài tại trang quản trị hoặc gọi:

```text
POST /api/v1/problems/import-zip
Content-Type: multipart/form-data
Field: file
```

### Cấu trúc ZIP

```text
problem-package.zip
├── problem.json
└── tests/
    ├── 01.in
    ├── 01.out
    ├── 02.in
    ├── 02.out
    ├── hidden_03.in
    └── hidden_03.out
```

### `problem.json`

```json
{
  "title": "Tính tổng hai số",
  "description": "Nhập hai số nguyên a, b và in ra a + b.",
  "time_limit": 1.0,
  "memory_limit": 256,
  "tags": "Cơ bản, Toán học, Dễ"
}
```

### Quy ước test case

- Mỗi file `.in` phải có file `.out` cùng tên.
- File phải nằm trong thư mục `tests/`.
- Tên chứa `hidden` hoặc `secret` sẽ được đánh dấu là test case ẩn.
- Dữ liệu được đọc dưới dạng UTF-8.
- Test case thiếu file `.out` tương ứng sẽ bị bỏ qua.

## API chính

Base URL mặc định:

```text
http://localhost:8000/api/v1
```

### Xác thực

| Method | Endpoint | Mô tả | Quyền |
|---|---|---|---|
| POST | `/auth/register` | Đăng ký user mới | Công khai |
| POST | `/auth/login` | Đăng nhập và nhận JWT | Công khai |
| GET | `/users/me` | Lấy user hiện tại | Đã đăng nhập |

> `/auth/login` dùng `application/x-www-form-urlencoded` theo chuẩn `OAuth2PasswordRequestForm`, không dùng JSON.

### Người dùng

| Method | Endpoint | Mô tả | Quyền |
|---|---|---|---|
| GET | `/users` | Danh sách user | Admin |
| POST | `/users` | Tạo user | Admin |
| PUT | `/users/{user_id}` | Sửa user | Admin |
| DELETE | `/users/{user_id}` | Xóa user | Admin |

### Đề bài và test case

| Method | Endpoint | Mô tả | Quyền |
|---|---|---|---|
| GET | `/problems` | Danh sách đề, hỗ trợ query `tag` | Công khai |
| POST | `/problems` | Tạo đề | Admin |
| POST | `/problems/import-zip` | Import đề từ ZIP | Admin |
| GET | `/problems/{problem_id}` | Chi tiết đề | Công khai |
| PUT | `/problems/{problem_id}` | Sửa đề và có thể thay toàn bộ test case | Admin |
| DELETE | `/problems/{problem_id}` | Xóa đề | Admin |
| POST | `/problems/{problem_id}/testcases` | Thêm test case | Admin |
| DELETE | `/problems/testcases/{testcase_id}` | Xóa test case | Admin |

Người dùng thường chỉ thấy test case công khai. Admin/Super Admin có thể xem cả test case ẩn khi lấy chi tiết đề.

### Bài nộp

| Method | Endpoint | Mô tả | Quyền |
|---|---|---|---|
| POST | `/submissions` | Tạo bài nộp và đẩy task chấm | Đã đăng nhập |
| GET | `/submissions` | Danh sách bài nộp | Đã đăng nhập |
| GET | `/submissions/{submission_id}` | Chi tiết bài nộp | Chủ bài/Admin |
| GET | `/problems/{problem_id}/submissions` | Lịch sử cá nhân theo đề | Đã đăng nhập |

Student chỉ thấy bài nộp của chính mình. Admin/Super Admin thấy toàn bộ danh sách.

### Ranking

| Method | Endpoint | Mô tả | Quyền |
|---|---|---|---|
| GET | `/rankings` | Bảng xếp hạng | Công khai |

### Bài viết

| Method | Endpoint | Mô tả | Quyền |
|---|---|---|---|
| GET | `/articles` | Danh sách bài viết | Công khai |
| GET | `/articles/{article_id}` | Chi tiết bài viết | Công khai |
| POST | `/articles` | Tạo bài viết | Admin |
| PUT | `/articles/{article_id}` | Sửa bài viết | Admin |
| DELETE | `/articles/{article_id}` | Xóa bài viết | Admin |

Hiện frontend chưa tích hợp đầy đủ API bài viết.

## Phân quyền

Hệ thống có ba vai trò:

| Vai trò | Khả năng chính |
|---|---|
| `STUDENT` | Xem đề, nộp bài, xem bài nộp cá nhân và ranking |
| `ADMIN` | Có quyền quản lý đề và chỉ quản lý tài khoản STUDENT |
| `SUPER_ADMIN` | Có toàn bộ quyền admin và có thể quản lý các vai trò cao hơn |

Các kiểm tra quyền quan trọng được thực hiện tại backend, không chỉ dựa vào route guard của frontend.

## Trạng thái bài nộp

| Trạng thái | Ý nghĩa |
|---|---|
| `PENDING` | Đang chờ Celery Worker xử lý |
| `AC` | Accepted — đúng toàn bộ test case |
| `WA` | Wrong Answer hoặc một số runtime error được quy về WA |
| `TLE` | Time Limit Exceeded |
| `MLE` | Memory Limit Exceeded |
| `CE` | Compilation Error; lỗi gọi Judge0 hiện cũng có thể bị trả về CE |
| `SYSTEM_ERROR` | Lỗi hệ thống, ví dụ đề chưa có test case |

## Phân tích các module quan trọng

### `backend/app/api/dependencies.py`

- Đọc Bearer token từ header `Authorization`.
- Giải mã JWT và lấy user bằng `sub`.
- Cung cấp ba dependency chính:
  - `get_current_user`
  - `require_admin`
  - `require_super_admin`
- Có phiên bản optional để endpoint chi tiết đề vẫn hoạt động khi khách chưa đăng nhập.

### `backend/app/services/sandbox.py`

- Base64-encode source code, stdin và expected output trước khi gửi Judge0.
- Chuyển memory limit từ MB sang KB.
- Gọi Judge0 với `wait=true` để worker nhận kết quả đồng bộ cho từng test.
- Chuyển status ID của Judge0 thành enum nội bộ.
- Chuyển bộ nhớ Judge0 từ KB về MB.

### `backend/app/worker/tasks.py`

Đây là trung tâm của hệ thống chấm:

- Celery task đồng bộ tạo event loop để chạy logic async.
- Tải quan hệ `problem`, `test_cases` và `user` bằng eager loading.
- Chạy test case tuần tự.
- Dừng tại lỗi đầu tiên.
- Lưu max execution time và max memory.
- Gọi AI Agent cho lỗi không phải AC/CE.
- Tính lại ranking của user sau khi lưu kết quả.

### `backend/app/services/ai_agent.py`

- Gọi endpoint `/chat/completions` tương thích OpenAI.
- System prompt yêu cầu phản hồi bằng tiếng Việt.
- Prompt cấm model cung cấp code lời giải và chỉ cho phép phân tích logic/độ phức tạp.
- Khi LLM lỗi hoặc timeout, hệ thống trả về một thông báo dự phòng thay vì làm task bị crash.

### `frontend/src/contexts/AuthContext.jsx`

- Lưu JWT trong `localStorage`.
- Kiểm tra `exp` trong payload khi tải lại trang.
- Gọi `/users/me` để lấy role và dữ liệu user thật từ backend.
- Xóa token nếu token hết hạn hoặc xác minh thất bại.

### `frontend/src/services/api.js`

- Tạo Axios instance dùng `VITE_API_URL`.
- Request interceptor tự động gắn:

```http
Authorization: Bearer <token>
```

### `frontend/src/App.jsx`

- Khai báo router cho các trang chính.
- Dùng `AdminRoute` để chặn giao diện quản trị nếu user không phải ADMIN/SUPER_ADMIN.
- Backend vẫn là lớp bảo vệ cuối cùng cho mọi thao tác nhạy cảm.

## Các điểm cần lưu ý

### 1. Sai khác tên database mặc định

`docker-compose.yml` tạo database `intellijudge_db`, trong khi `backend/.env.example` đang dùng `intellijudge`. Khi chạy bằng Docker Compose, nên dùng:

```env
DATABASE_URL="mysql+aiomysql://root:rootpassword@localhost:3306/intellijudge_db"
```

### 2. Judge0 và LLM chưa nằm trong Docker Compose

`docker compose up -d` hiện chỉ chạy MySQL và Redis. Judge0 là bắt buộc cho chấm bài; LLM là tùy chọn cho gợi ý AI.

### 3. Dữ liệu mô phỏng ở frontend

- `ProblemTable` tự tạo độ khó, số lượt nộp và số bài đạt nếu backend không trả các trường này.
- `ProblemSidebar` chứa tag tĩnh và biểu đồ kỹ năng placeholder.
- Trang `contests` mới là placeholder.
- API bài viết đã có ở backend nhưng chưa được frontend sử dụng.

### 4. Bảo mật production

Trước khi đưa lên production cần:

- Đổi `JWT_SECRET_KEY` thành chuỗi bí mật mạnh.
- Không sử dụng tài khoản `admin/admin123`.
- Không để `allow_origins=["*"]` trong CORS.
- Bật HTTPS.
- Giới hạn kích thước code và file ZIP upload.
- Kiểm tra chống ZIP bomb/path traversal kỹ hơn.
- Không log dữ liệu nhạy cảm.
- Cân nhắc lưu token bằng cookie HttpOnly thay vì `localStorage`.
- Kiểm tra chính sách dữ liệu vì source code và test case lỗi có thể được gửi tới dịch vụ LLM.

### 5. Database migration

Dự án hiện dùng script `create_tables.py` và một migration thủ công `add_tags_column.py`; chưa tích hợp Alembic. Nên thêm Alembic để quản lý version schema an toàn.

### 6. Xử lý lỗi Judge0

`Judge0Service` hiện chuyển lỗi kết nối Judge0 thành `CE`. Điều này có thể khiến lỗi hạ tầng bị hiểu nhầm là lỗi code. Nên ánh xạ lỗi kết nối sang `SYSTEM_ERROR`.

### 7. Hiệu năng

- Worker chạy test case tuần tự và gọi Judge0 với `wait=true` cho từng test.
- Endpoint ranking chạy thêm một câu query đếm submission cho từng user, tạo mẫu N+1 query.
- Mỗi lần chấm xong hệ thống đọc toàn bộ submission của user để tính lại ranking.

Các cách tối ưu có thể gồm batch submission Judge0, aggregate query, cache ranking hoặc cập nhật ranking theo delta.

### 8. Kiểm thử

Repository hiện chưa có thư mục test tự động cho frontend/backend. Nên bổ sung:

- Unit test cho security, status mapping và ranking.
- API integration test với database test.
- Mock Judge0 và LLM.
- Frontend component/e2e test.

## Hướng phát triển

- Thêm Alembic migration.
- Thêm Docker Compose đầy đủ cho backend, frontend, Judge0 và LLM.
- Thêm cuộc thi, thời gian bắt đầu/kết thúc và scoreboard theo contest.
- Thêm trường difficulty và thống kê submission thật vào backend.
- Thêm bài viết/hướng dẫn vào frontend.
- Thêm WebSocket hoặc Server-Sent Events để cập nhật trạng thái chấm realtime thay vì polling.
- Thêm retry, timeout policy và dead-letter handling cho Celery.
- Tách runtime error thành trạng thái riêng.
- Chấm song song test case khi phù hợp.
- Thêm rate limit, audit log và quản lý refresh token.
- Thêm CI để lint, test và build tự động.

## Kiểm tra mã nguồn

Backend đã được kiểm tra cú pháp bằng:

```bash
python -m compileall -q backend
```

Kết quả tại thời điểm tạo README: không phát hiện lỗi cú pháp Python.

---

IntelliJudge là nền tảng phù hợp để học kiến trúc full-stack, xử lý task nền, sandbox thực thi code, xác thực JWT và tích hợp LLM vào hệ thống giáo dục.
