# Day 10 Reliability Report

## 1. Architecture summary

Hệ thống có một lớp gateway để định tuyến các request đến provider thích hợp, sử dụng Circuit Breaker để bảo vệ các provider khỏi tình trạng quá tải khi bị lỗi. Ngoài ra, hệ thống còn áp dụng một lớp semantic cache (dựa trên bộ nhớ/Redis) để tăng tốc độ phản hồi và tiết kiệm chi phí. Khi primary provider gặp sự cố, hệ thống sẽ tự động fallback sang backup provider, hoặc trả về một thông báo tĩnh nếu cả hai đều thất bại.

```text
User Request
    |
    v
[Gateway] ---> [Cache check] ---> HIT? return cached
    |                                 |
    v                                 v MISS
[Circuit Breaker: Primary] -------> Provider A
    |  (OPEN? skip)
    v
[Circuit Breaker: Backup] --------> Provider B
    |  (OPEN? skip)
    v
[Static fallback message]
```

## 2. Configuration

| Setting | Value | Reason |
|---|---:|---|
| failure_threshold | 3 | Tránh tình trạng ngắt mạch quá sớm nếu chỉ có 1-2 request lỗi mạng tạm thời, nhưng đủ nhạy để ngắt khi lỗi liên tục. |
| reset_timeout_seconds | 2 | Đủ lâu để provider phục hồi, nhưng không quá lâu để tránh tăng thời gian fallback quá mức. |
| success_threshold | 1 | Chỉ cần 1 request thành công để xác nhận provider đã phục hồi (probe request). |
| cache TTL | 300 | Giữ dữ liệu trong 5 phút để tăng tốc, tránh giữ quá lâu khiến thông tin bị lỗi thời. |
| similarity_threshold | 0.92 | Ngưỡng cosine similarity n-gram đủ cao để tránh false-hits (trả lời sai) nhưng vẫn bao phủ được các câu hỏi tương tự nhau. |
| load_test requests | 100 | Số lượng request trong mỗi kịch bản chaos để có mẫu đủ lớn đánh giá các chỉ số. |

## 3. SLO definitions

Define your target SLOs and whether your system meets them:

| SLI | SLO target | Actual value | Met? |
|---|---|---:|---|
| Availability | >= 99% | 99.0% | Yes |
| Latency P95 | < 2500 ms | 318.27 ms | Yes |
| Fallback success rate | >= 95% | 96.81% | Yes |
| Cache hit rate | >= 10% | 58.67% | Yes |
| Recovery time | < 5000 ms | 2249.29 ms | Yes |

## 4. Metrics

Dữ liệu được trích xuất từ `reports/metrics.json` sau khi chạy `run_chaos`.

| Metric | Value |
|---|---:|
| availability | 0.99 |
| error_rate | 0.01 |
| latency_p50_ms | 281.52 ms |
| latency_p95_ms | 318.27 ms |
| latency_p99_ms | 320.34 ms |
| fallback_success_rate | 0.9681 |
| cache_hit_rate | 0.5867 |
| estimated_cost_saved | $0.176 |
| circuit_open_count | 12 |
| recovery_time_ms | 2249.29 ms |

## 5. Cache comparison

Dựa trên kết quả mô phỏng, Semantic Cache mang lại hiệu quả rất lớn về cả chi phí và thời gian.

| Metric | Without cache | With cache | Delta |
|---|---:|---:|---|
| latency_p50_ms | ~220 ms | 281.52 ms | (phụ thuộc cache hit/miss) |
| latency_p95_ms | ~350 ms | 318.27 ms | Giảm độ trễ đuôi dài (Tail latency) |
| estimated_cost | $0.223 | $0.047 | Tiết kiệm ~78% chi phí |
| cache_hit_rate | 0 | 58.67% | +58.67% |

## 6. Redis shared cache

Explain why shared cache matters for production:

- **Why in-memory cache is insufficient for multi-instance deployments**: Khi scale hệ thống với nhiều instance gateway, in-memory cache của mỗi instance là độc lập. Một truy vấn đã được cache ở instance A sẽ gặp cache-miss nếu được định tuyến vào instance B, dẫn đến lãng phí resource và call LLM không cần thiết.
- **How `SharedRedisCache` solves this**: Redis cung cấp một kho lưu trữ chung tập trung cho tất cả các instance. Bất kỳ instance nào tính toán và lưu cache đều có thể được chia sẻ cho toàn bộ cluster, tối ưu hit rate đáng kể.

## 7. Chaos scenarios

| Scenario | Expected behavior | Observed behavior | Pass/Fail |
|---|---|---|---|
| primary_timeout_100 | All traffic fallback to backup, circuit opens | Mạch mở do lỗi liên tục, các request tự động nhảy sang Backup Provider (thành công cao). | Pass |
| primary_flaky_50 | Circuit oscillates, mix of primary and fallback | Mạch nhảy liên tục giữa OPEN và HALF_OPEN, một lượng vừa phải request được phục vụ bởi primary, còn lại fallback. | Pass |
| all_healthy | All requests via primary, no circuit opens | Không có mạch nào mở, 100% qua Primary Provider hoặc Cache. | Pass |

## 8. Failure analysis

**Điểm yếu còn tồn tại và cách khắc phục trước khi lên production:**
- **Thử nghiệm mở rộng**: Hệ thống hiện tại có reset_timeout_seconds tĩnh. Nếu provider chưa khắc phục được lỗi trong thời gian dài, gateway vẫn liên tục mở HALF_OPEN gây lãng phí.
- **Cách khắc phục**: Triển khai thuật toán Exponential Backoff cho `reset_timeout_seconds` thay vì dùng một giá trị cố định (VD: 2s -> 4s -> 8s) để không làm dồn dập các request thử nghiệm lên một dịch vụ đang sập (thundering herd).

## 9. Next steps

List 2-3 concrete improvements you would make:

1. Thêm thuật toán Exponential Backoff và Jitter cho Circuit Breaker Timeout.
2. Thêm tính năng cấu hình Circuit Breaker lưu trữ trạng thái trên Redis (Distributed Circuit Breaker) để các instance đồng bộ trạng thái sập/mở của provider.
3. Bổ sung Budget Limit: Tự động từ chối phục vụ hoặc luôn sử dụng Static Fallback khi chi phí API vượt quá ngân sách trong một ngày.
