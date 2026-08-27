# Nguyễn Thị Lý -2A202601962 
# Day 10 Reliability Report

## 1. Architecture summary

The Reliability Gateway (`ReliabilityGateway`) coordinates three resiliency layers to handle LLM requests, reduce latency, and lower API costs:

1. **Semantic Cache Layer (`ResponseCache` / `SharedRedisCache`)**: Intercepts requests before calling providers. Evaluates query similarity using n-gram cosine similarity (word tokens + char 3-grams) and applies privacy (`_is_uncacheable`) and false-hit guardrails (`_looks_like_false_hit`). Hits return immediately with `latency_ms=0` and zero cost.
2. **Circuit Breaker Pattern (`CircuitBreaker`)**: Protects each provider independently (`CLOSED` -> `OPEN` -> `HALF_OPEN`). When `failure_count >= failure_threshold`, the circuit trips to `OPEN`, denying requests fast to prevent cascading failures and allow the provider to recover.
3. **Provider Fallback Chain & Static Fallback (`ReliabilityGateway`)**: Iterates through ordered providers (`primary`, `backup`). If a provider raises `ProviderError` or `CircuitOpenError`, the gateway catches the exception, logs it, and falls back to the next provider. If all providers fail, it returns a static fallback response (`static_fallback`).

```
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached (route: cache_hit:score)
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A (primary)
    |  (OPEN / ProviderError? fallback)
    v
[Circuit Breaker: Backup] --------> Provider B (backup)
    |  (OPEN / ProviderError? fallback)
    v
[Static fallback message] (route: static_fallback)
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Đặt ngưỡng lỗi bằng 3 trong `CircuitBreaker` để tránh ngắt cầu dao quá sớm do 1-2 lỗi tức thời (transient errors), đảm bảo chỉ ngắt sang `OPEN` khi provider thực sự có lỗi liên tiếp 3 lần. |
| reset_timeout_seconds | 2 | Trong `CircuitBreaker.allow_request()`, trạng thái `OPEN` sẽ từ chối request cho đến khi đủ 2 giây (`time.monotonic() - opened_at >= 2.0`). Khoảng thời gian này giúp provider có đủ thời gian phục hồi trước khi nhận request thăm dò ở trạng thái `HALF_OPEN`. |
| success_threshold | 1 | Trong `CircuitBreaker.record_success()`, khi ở trạng thái `HALF_OPEN`, chỉ cần 1 request thăm dò thành công (`success_count >= 1`) là mạch ngay lập tức chuyển về `CLOSED` với lý do `"probe_success"`, giúp khôi phục lưu lượng nhanh chóng. |
| cache TTL | 300 | Trong `ResponseCache` và `SharedRedisCache`, TTL 300 giây (5 phút) loại bỏ các entry hết hạn (`now - created_at > 300`), vừa đủ để phục vụ các câu hỏi lặp lại trong đợt cao điểm vừa đảm bảo làm tươi dữ liệu định kỳ. |
| similarity_threshold | 0.92 | Ngưỡng cosine similarity trong `ResponseCache.get()` đặt ở 0.92 để chỉ chấp nhận các câu hỏi có mức độ trùng lặp token và 3-gram rất cao. Mức 0.92 ngăn ngừa khớp sai (ví dụ: các câu hỏi có n-gram overlap ~0.69 như "circuit breaker pattern" vs "circuit breaker design"). |
| load_test requests | 100 | Cấu hình `load_test.requests` trong `configs/default.yaml` đặt 100 requests/scenario. Chạy qua 3 scenario sinh ra tổng cộng 300 requests, đủ mẫu dữ liệu để đo đạc chính xác các chỉ số P50/P95/P99 latency và availability. |

## 3. SLO definitions

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.00% | YES |
| Latency P95 | < 2500 ms | 314.33 ms | YES |
| Fallback success rate | >= 95% | 95.83% | YES |
| Cache hit rate | >= 10% | 67.67% | YES |
| Recovery time | < 5000 ms | 2318.40 ms | YES |

## 4. Metrics

Dữ liệu thực tế trích xuất trực tiếp từ file `reports/metrics.json` (kết quả chạy simulation với cấu hình default + Redis cache):

| Metric | Value |
|---|---:|
| availability | 0.9900 |
| error_rate | 0.0100 |
| latency_p50_ms | 279.52 |
| latency_p95_ms | 314.33 |
| latency_p99_ms | 318.35 |
| fallback_success_rate | 0.9583 |
| cache_hit_rate | 0.6767 |
| estimated_cost | $0.039578 |
| estimated_cost_saved | $0.203000 |
| circuit_open_count | 8 |
| recovery_time_ms | 2318.40 |

## 5. Cache comparison

Kết quả đo đạc thực tế khi chạy 300 requests giữa 2 chế độ (đã xóa sạch Redis qua `FLUSHDB` trước khi đo):

| Metric | Without cache (`metrics_no_cache.json`) | With cache (`metrics.json`) | Delta |
|---|---:|---:|---|
| latency_p50_ms | 276.54 ms | 279.52 ms | +2.98 ms |
| latency_p95_ms | 315.81 ms | 314.33 ms | -1.48 ms |
| estimated_cost | $0.124772 | $0.039578 | -$0.085194 (-68.28%) |
| cache_hit_rate | 0.0% | 67.67% | +67.67% |

*Nhận xét dựa trên dữ liệu*: Việc bật Semantic Cache giúp giảm 68.28% chi phí ước tính ($0.1248 xuống $0.0396) và tiết kiệm $0.2030 nhờ phục vụ 67.67% số request từ cache thay vì gọi provider.

## 6. Redis shared cache

### Why shared cache matters for production

- **Hạn chế của In-memory cache**: Trong môi trường multi-instance (ví dụ 3 pod Gateway chạy phía sau Load Balancer), `ResponseCache` lưu trong bộ nhớ RAM cục bộ của từng process. Nếu Pod A đã lưu response vào cache, Pod B và Pod C vẫn không thấy dữ liệu này, dẫn đến việc gọi lặp lại provider không cần thiết và tỷ lệ cache hit không đồng nhất giữa các pod.
- **Giải pháp của `SharedRedisCache`**: Tất cả các process Gateway cùng kết nối đến 1 instance Redis chung và thao tác trên namespace key `rl:cache:*`. Khi một query được bất kỳ pod nào lưu vào Redis, tất cả các pod khác lập tức truy xuất được ngay.

### Evidence of shared state

Xác nhận thực tế qua test `test_shared_state_across_instances` trong `tests/test_redis_cache.py`: 2 đối tượng `SharedRedisCache` độc lập (`cache1` và `cache2`) đọc/ghi dữ liệu đồng bộ qua Redis:

```python
cache1 = SharedRedisCache(redis_url, ttl_seconds=60, similarity_threshold=0.8)
cache2 = SharedRedisCache(redis_url, ttl_seconds=60, similarity_threshold=0.8)
cache1.set("what is a circuit breaker", "A design pattern")
res, score = cache2.get("what is a circuit breaker")
assert res == "A design pattern"
```

### Redis CLI output

Dữ liệu thực tế kiểm tra keys trong Redis container sau khi chạy chaos scenario:

```bash
$ docker compose exec redis redis-cli KEYS "rl:cache:*"
rl:cache:9e413fd814eb
rl:cache:3dab98c0e49e
rl:cache:dacb2b833659
rl:cache:4fc3c69b9376
rl:cache:734852f3cf4a
rl:cache:3936614ac4c2
rl:cache:da61fb49b4f6
rl:cache:095946136fea
rl:cache:d354658dc020
rl:cache:844ef0143a5c
rl:cache:fff10da1c72c
rl:cache:0bc3b1acf73d
rl:cache:98332d0d1c9c
```

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | Primary có fail_rate=1.0, tất cả request phải chuyển sang backup | Cầu dao primary chuyển sang OPEN sau 3 lần thất bại. 100% request không bị drop mà được route sang backup. | PASS |
| primary_flaky_50 | Primary có fail_rate=0.5, cầu dao liên tục mở/đóng | Cầu dao primary liên tục chuyển đổi giữa OPEN, HALF_OPEN và CLOSED. Backup xử lý các request bị primary từ chối. | PASS |
| all_healthy | Cả primary và backup đều bình thường | Tất cả các request không có trong cache đều được xử lý thành công bởi primary provider. 0 lần mở cầu dao. | PASS |

## 8. Failure analysis

Dựa trên việc kiểm tra codebase (`src/reliability_lab/`):

1. **Trạng thái CircuitBreaker lưu cục bộ trong RAM của process**:
   - Trong `src/reliability_lab/gateway.py` và `circuit_breaker.py`, `self.breakers` là một dictionary lưu trong bộ nhớ RAM của Python process.
   - Khi triển khai multi-pod, nếu provider `primary` bị lỗi, Pod 1 sẽ đếm đủ 3 lỗi và chuyển cầu dao sang `OPEN`. Tuy nhiên Pod 2 và Pod 3 không biết điều này và vẫn giữ trạng thái `CLOSED`, tiếp tục gửi request lỗi vào provider `primary` cho đến khi từng pod tự đếm đủ 3 lỗi.
   - *Cách khắc phục*: Đồng bộ trạng thái ngắt mạch (Circuit State) lên Redis (dùng Redis Pub/Sub hoặc Hash key chung) để khi 1 pod ngắt cầu dao, tất cả các pod khác lập tức chuyển trạng thái theo.

2. **Chưa có fallback khi Redis bị sự cố**:
   - Trong `src/reliability_lab/cache.py`, các lệnh `self._redis.hget`, `self._redis.scan_iter`, `self._redis.hset` gọi trực tiếp Redis client mà không có block `try-except`.
   - Nếu Redis server bị sập hoặc mất kết nối mạng, `SharedRedisCache.get()` sẽ văng exception làm gián đoạn request thay vì tự động bỏ qua cache để gọi provider.
   - *Cách khắc phục*: Bọc các thao tác Redis trong block `try-except`, nếu có lỗi kết nối Redis thì ghi log cảnh báo và trả về `(None, 0.0)` để Gateway tiếp tục gọi provider an toàn.

## 9. Next steps

1. **Đồng bộ trạng thái Circuit Breaker qua Redis Pub/Sub**: Đưa trạng thái ngắt mạch lên Redis để tất cả các gateway process chia sẻ chung trạng thái ngắt/mở cầu dao theo thời gian thực.
2. **Graceful Fallback cho Cache Layer**: Thêm xử lý ngoại lệ khi thao tác với Redis trong `SharedRedisCache`, cho phép hệ thống tự động fallback sang in-memory cache hoặc bypass cache khi Redis gặp sự cố.
3. **Tối ưu hóa truy vấn Similarity bằng Vector Index**: Thay thế cơ chế duyệt quét toàn bộ key (`scan_iter`) trong `SharedRedisCache.get()` bằng RediSearch Vector Search (HNSW Index) để giữ latency tra cứu ở mức thấp khi số lượng cache entry lên tới hàng chục ngàn câu.
