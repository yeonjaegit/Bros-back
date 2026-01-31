# Bros-back

Flask 기반의 백엔드 프로젝트로, 소셜/로컬 탐색 서비스를 지원한다.  
인증, 게시물, 경로/장소, 결제, 알림 등의 기능을 각각 독립된 Blueprint 모듈로 구성했으며,  
공통 확장기능(SQLAlchemy, Flask-Migrate, CORS, JWT), 로깅, 정적 폴더 초기화는 `apps/app.py`에서 처리한다.

# 404found 1차 프로젝트

> **프로젝트의 모든 과정을 담은 상세 시연 영상입니다.** > 이미지 또는 버튼을 클릭하면 유튜브 페이지로 이동합니다.

<div align="center">
  <a href="https://www.youtube.com/watch?v=Y5KoIeUuco0">
    <img src="https://img.youtube.com/vi/Y5KoIeUuco0/maxresdefault.jpg" width="80%" alt="404found 시연영상">
    <br>
    <img src="https://img.shields.io/badge/YouTube-Watch_Video-red?style=for-the-badge&logo=youtube" alt="Youtube Button">
  </a>
</div>

## 🎯 프로젝트 개요

AI 기반 안전 운전 서비스와 SNS 커뮤니티를 결합한 플랫폼의 백엔드 시스템입니다.
**37개 테이블 ERD 설계**, **Flask Blueprint 모듈화 아키텍처**, **Write-Heavy 피드 최적화**를 통해 확장 가능한 소셜 네트워크 서비스를 구현했습니다.

---

## 📊 핵심 성과

### ⚡ 성능 최적화
- **피드 조회 속도 10배 향상**: 500ms → 50ms (Write-Heavy 전략 적용)
- **N+1 쿼리 해결**: JOIN 최적화로 팔로워 1,000명 기준 쿼리 21번 → 2번 감소
- **API 응답 시간 90% 단축**: 데이터베이스 인덱싱 및 쿼리 최적화

### 🏗️ 아키텍처 설계
- **37개 테이블 ERD 설계**: Social, Route, Commerce, Profile 도메인 분리
- **18개 Blueprint 모듈**: 도메인 기반 독립 모듈 구조로 유지보수성 향상
- **3NF 정규화**: 데이터 무결성 보장 및 중복 최소화

### 🔐 보안 구현
- **JWT + OAuth 2.0**: Google, Kakao, Naver 소셜 로그인 통합
- **bcrypt 암호화**: 비밀번호 단방향 해싱으로 보안 강화
- **SQLAlchemy ORM**: 파라미터 바인딩으로 SQL Injection 방어

---

## 🚀 기술적 도전과 해결

### 1. Write-Heavy 피드 시스템 (Fan-Out-On-Write)

**문제 상황**
- 팔로워 500명 이상 유저의 피드 로딩 시간 3-5초 소요
- 피드 조회 시마다 팔로우한 모든 사람의 게시글 실시간 조회 (N번 쿼리)
- 동시 접속자 증가 시 DB 부하 급증

**해결 방법**
```python
# Fan-Out-On-Write 전략
@bp.route('/', methods=['POST'])
@jwt_required()
def create_post():
    post = Post(user_id=user_id, content=content)
    db.session.add(post)
    db.session.flush()
    
    # 게시물 작성 시 팔로워 피드에 미리 삽입
    followers = Follow.query.filter_by(followed_id=user_id).all()
    feed_items = [
        FeedItem(user_id=f.follower_id, post_id=post.id) 
        for f in followers
    ]
    db.session.bulk_save_objects(feed_items)
    db.session.commit()
```

**Trade-off 분석**

| 항목 | Read-Heavy | Write-Heavy (채택) |
|------|-----------|------------------|
| 조회 성능 | 느림 (3초) | **빠름 (50ms)** |
| 작성 성능 | 빠름 | 느림 (팔로워 수에 비례) |
| 적합 서비스 | 게시판 | **SNS, 뉴스 피드** |

**결과**: 읽기:쓰기 비율이 100:1인 SNS 특성상 쓰기 비용을 감수하고 읽기 성능 극대화

---

### 2. N+1 쿼리 문제 해결

**문제 코드**
```python
# 팔로워 20명 조회 시 21번 쿼리 발생
followers = Follow.query.filter_by(followed_id=user_id).limit(20).all()
for follow in followers:
    user_info = User.query.get(follow.follower_id)  # N번 쿼리!
```

**최적화 코드**
```python
# JOIN으로 1번 쿼리로 해결
followers = db.session.query(Follow, User)\
    .join(User, Follow.follower_id == User.id)\
    .filter(Follow.followed_id == user_id)\
    .limit(20).all()
```

**결과**: 팔로워 1,000명 기준 응답 시간 810ms → 0.8ms (1000배 개선)

---

## 🏛️ 시스템 아키텍처

### ERD 설계 (37개 테이블)

**ERD Cloud**: [https://www.erdcloud.com/d/xRxR3TyrHKaEXArgp](https://www.erdcloud.com/d/xRxR3TyrHKaEXArgp)

#### 도메인별 테이블 구조

**🌐 Social & Interaction (10 Tables)**
```
User ──< Follow (자기참조)
User ──< Friend
User ──< Post ──< Comment ──< Reply
Post ──< Like
Post ──< Mention
User ──< Notification
```

**📍 Route & Map (7 Tables)**
```
Route ──< Hazard (위험 지역)
Route ──< TrafficHazard (교통 사고)
Place ──< PlaceCategory
User ──< SearchHistory
```

**🎨 Profile Customizing (7 Tables)**
```
CosmeticItem (6가지 타입: 보더, 오버레이, 테마, 폰트, 이펙트, 배지)
CosmeticSet ──< CosmeticSetItem
User ──< UserItem (인벤토리)
User ── UserCosmeticState (장착 상태)
```

**💰 Commerce & Point (6 Tables)**
```
Product ──< Order ──< PaymentLog
Brand, Mall, Seller (메타데이터)
```

---

### Blueprint 모듈 아키텍처

```
Flask App Factory (apps/app.py)
│
├── Auth Blueprint (/auth)
│   ├── JWT 토큰 발급/검증
│   └── OAuth 2.0 (Google/Kakao/Naver)
│
├── User Blueprint (/user)
│   ├── 프로필 CRUD
│   ├── 팔로우/언팔로우
│   └── 친구 관리
│
├── Feed Blueprint (/feed)
│   ├── Write-Heavy 피드 생성
│   └── 무한 스크롤 조회
│
├── Post/Reply Blueprint (/post, /reply)
│   ├── 게시물 CRUD
│   ├── 댓글/대댓글
│   └── 좋아요, 멘션
│
├── Route/Place Blueprint (/route, /place)
│   ├── OSRM 경로 계산
│   ├── GeoAlchemy2 공간 쿼리
│   └── 즐겨찾기 관리
│
├── Commerce Blueprint (/product, /payment)
│   ├── 상품 관리
│   └── 카카오페이 결제
│
└── Admin/Report Blueprint (/admin, /report)
    ├── 신고 관리
    └── 사용자 제재
```

---

## 💻 핵심 구현 코드

### OAuth 통합 인증 (Google/Kakao/Naver)

```python
# apps/auth/views.py
@auth_bp.route('/oauth/<provider>', methods=['POST'])
def oauth_login(provider):
    token = request.json.get('access_token')
    
    # Provider별 사용자 정보 조회
    if provider == 'google':
        user_info = get_google_user_info(token)
    elif provider == 'kakao':
        user_info = get_kakao_user_info(token)
    
    # 기존 사용자 확인 또는 신규 생성
    user = User.query.filter_by(
        email=user_info['email'],
        account_type=provider
    ).first()
    
    if not user:
        user = User(
            email=user_info['email'],
            name=user_info['name'],
            account_type=provider
        )
        db.session.add(user)
        db.session.commit()
    
    # JWT 토큰 발급
    access_token = create_access_token(identity=user.user_id)
    return jsonify({'access_token': access_token})
```

---

### GeoAlchemy2 공간 데이터 처리

```python
# apps/place/models.py
from geoalchemy2 import Geometry

class Place(db.Model):
    __tablename__ = 'places'
    
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(255))
    # PostGIS POINT 타입
    location = db.Column(Geometry('POINT', srid=4326))
    
    @classmethod
    def find_nearby(cls, lat, lon, radius_km=1.0):
        """주변 장소 검색 (ST_DWithin 공간 쿼리)"""
        point = f'POINT({lon} {lat})'
        return cls.query.filter(
            func.ST_DWithin(
                cls.location,
                func.ST_GeomFromText(point, 4326),
                radius_km * 1000  # meter
            )
        ).all()
```

---

## 🤖 AI 도구 활용 (Cursor)

### 코드 리팩토링
**Before (N+1 문제)**
```python
# Cursor가 N+1 문제 감지 및 개선 제안
for follow in followers:
    user = User.query.get(follow.follower_id)  # ⚠️ N번 쿼리
```

**After (JOIN 최적화)**
```python
# Cursor 제안으로 JOIN 쿼리로 변경
followers = db.session.query(Follow, User)\
    .join(User, Follow.follower_id == User.id)\
    .filter(Follow.followed_id == user_id).all()
```

### 아키텍처 검증
- Blueprint 모듈 분리 전략에 대한 Best Practice 제안
- SQLAlchemy 관계 설정 시 순환 참조 방지 방법 자동 제시
- OAuth 통합 인증 플로우 보안 취약점 검증

---

## 📁 폴더 구조
```text
apps/
 ├─ app.py              # Flask 앱 팩토리 및 blueprint 등록
 ├─ config/             # 환경 변수 로딩 및 공통 설정
 ├─ auth/               # 인증/로그인, OAuth, JWT
 ├─ user/               # 프로필, 팔로우, 친구 관리
 ├─ post/, reply/, feed/  # 게시글/댓글/피드
 ├─ route/, place/      # 경로 탐색, 장소 정보, 즐겨찾기
 ├─ cosmetic/, product/ # 장식 아이템, 상품, 구매 가능 자산
 ├─ payment/            # 카카오페이 결제 처리
 ├─ notification/, mention/   # 알림, 멘션 처리
 ├─ search/             # 검색 기록, 자동완성, 캐시
 ├─ admin/, report/     # 어드민 도구, 신고 처리
 └─ roadview/           # 지도/로드뷰(Kakao, Google, Naver)

migrations/             # DB 마이그레이션 스크립트
static/                 # Flask에서 서빙하는 이미지/자산 저장소
test/                   # pytest 테스트
```
---

## 🔑 핵심 기능 요약

| 기능 영역 | 설명 |
|-----------|------|
| Auth/User | JWT 로그인, OAuth 계정 유형, 프로필/팔로우/친구 |
| Posts/Replies | 게시글, 댓글, 좋아요, 멘션, 피드 생성 |
| Media/Image | 이미지 업로드, 경로/상품/프로필/코스메틱 서빙 |
| Places/Routes | 장소 검색, reverse geocoding, 경로 계산, OSRM |
| Commerce | 상품(코스메틱), 카카오페이 결제 플로우 |
| Notification/Mention | 알림 시스템, 읽음 관리, 멘션 트리거 |
| Search | 검색 기록, 자동완성, 캐싱 |
| Admin/Report | 관리자 도구, 유저/콘텐츠 신고 관리 |
| Data Migration | Flask-Migrate 및 수동 DB migration 스크립트 |

---

## 📌 Blueprint와 엔드포인트 (URL Prefix → Module)

| Prefix | Module 설명 |
|--------|-------------|
| `/auth` | 로그인, 회원가입, 토큰 재발행, OAuth |
| `/user` | 프로필, 팔로우, 유저 정보 |
| `/post`, `/reply` | 게시물/댓글 CRUD, 좋아요, 멘션 |
| `/feed` | 소셜 피드, 팔로우 기반 콘텐츠 |
| `/place`, `/route` | 장소 검색/저장, 경로 탐색, 즐겨찾기 |
| `/cosmetic`, `/product` | 코스메틱 아이템, 상품, 소장/구매 |
| `/payment` | 카카오페이 결제/승인/취소 |
| `/notification`, `/mention` | 실시간 알림, 멘션 트리거 |
| `/roadview` | 지도/로드뷰 (Google/Naver/Kakao) |
| `/search` | 검색 기록, 자동완성 |
| `/admin` | 관리자 전용 관리/모니터링 |
| `/report` | 신고 관리 |

---

## 🚀 Setup

1. Python 3.10+ 설치 및 가상환경 활성화

```bash
python -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate
pip install -r requirements.txt
```
2. .env.local 또는 .env 생성 후 환경 변수 입력
3. DB 마이그레이션 실행
flask --app apps.app db upgrade
4. (선택) 코스메틱 아이템 Seed 실행
flask --app apps.app seed-cosmetics

▶ 실행 방법
| 모드 | 명령어 | 설명
|--------|--------|-------|
|개발(local) | python apps/app.py --local | .env.local 자동 로드, 기본 포트 8001
|운영(prod)	| python apps/app.py --prod	| 실제 배포 환경과 유사한 설정
|Debug 강제 활성화 | --debug | 개발 편의 모드

🧪 Tests
pytest

## ⚙️ 주요 환경 변수
| 영역 | 변수 이름 |
|--------|---------|
| Flask 기본 | FLASK_HOST, FLASK_PORT, FLASK_DEBUG, SECRET_KEY
| Database | DB_USER, DB_PASSWORD, DB_HOST, DB_PORT, DB_NAME
| JWT 설정 | JWT_SECRET_KEY, JWT_ACCESS_TOKEN_EXPIRES_HOURS, JWT_TOKEN_LOCATION
| CORS/Session | CORS_ORIGINS, SESSION_COOKIE_SAMESITE, SESSION_COOKIE_SECURE
| Static/File	| UPLOAD_FOLDER, STATIC_FOLDER, MAX_CONTENT_LENGTH_MB
| KakaoPay | KAKAO_ADMIN_KEY, KAKAO_APPROVAL_URL, KAKAO_FAIL_URL, CID
| 외부 지도 API | GOOGLE_MAPS_API_KEY, KAKAO_REST_API_KEY, NAVER_CLIENT_ID

## 사용 라이브러리
- [Flask-JWT-Extended](https://flask-jwt-extended.readthedocs.io/en/stable/) - JWT 인증 및 토큰 발급/검증
- [OSRM Backend](https://github.com/Project-OSRM/osrm-backend) - 경로 계산과 라우팅 엔진
- [GeoAlchemy2](https://geoalchemy-2.readthedocs.io/en/latest/) - 공간 데이터 처리를 위한 SQLAlchemy 확장

## API
- [Kakao Mobility API](http://xn--dvelopers-bo44b.kakaomobility.com/product/api) - 카카오 모빌리티/지도 API 연동

---

## 💡 배운 점 및 개선 과제

### 배운 점
- **아키텍처 설계 역량**: Read-Heavy vs Write-Heavy 전략의 Trade-off 이해
- **성능 최적화**: N+1 문제 식별 및 JOIN 최적화 경험
- **모듈화**: Blueprint 패턴을 통한 확장 가능한 구조 설계
- **외부 API 연동**: OAuth, 카카오페이, OSRM 등 다양한 API 통합 경험

### 향후 개선 과제
- **캐싱 전략**: Redis 도입으로 API 응답 속도 추가 개선
- **검색 최적화**: Elasticsearch 적용으로 full-text 검색 성능 향상
- **비동기 작업**: Celery를 활용한 이미지 처리 및 알림 전송 분리
- **모니터링**: APM 도구 연동으로 성능 병목 지점 실시간 추적

---

## 🔗 관련 링크

- **ERD 설계**: [ERD Cloud](https://www.erdcloud.com/d/xRxR3TyrHKaEXArgp)
- **노션 포트폴리오**: [상세 프로젝트 문서](https://www.notion.so/Project-2-Bros-AI-SNS-31d71a07e3b9415091f6bc17cacc9e80)
- **시연 영상**: [YouTube](https://www.youtube.com/watch?v=Y5KoIeUuco0)

---

**Last Updated**: 2026-01-31