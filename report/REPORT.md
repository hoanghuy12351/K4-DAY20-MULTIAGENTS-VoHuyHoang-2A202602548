# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin nhóm và cấu hình

| Họ tên       | Mã sinh viên | Phần đóng góp |
| ------------ | ------------ | ------------- |
| Võ Huy Hoàng | 2A202602548  |               |

- Mô hình: `LAB_MODEL=openai:gpt-4o-mini`; `LAB_TEMPERATURE=0`; `recursion_limit=60` cho các lần chạy hiện có.
- Deep Agents `0.7.21`, Python `3.12.15` trong Docker Linux; các kết quả đầu tiên đã chạy trực tiếp trên Windows, `code-learn` và `data-learn` baseline sau đó chạy lại trong Docker.
- Đã quan sát ít nhất 8 lượt tác vụ học (6 kết quả hiện có và 2 kết quả baseline đã bị ghi đè). Chưa có ngân sách token riêng được ghi nhận.
- Commit của tag `freeze`:

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

Các giả thuyết dưới đây được lập trước khi mở kết quả tác vụ đánh giá. Dự đoán `baseline` có điểm trung bình đánh giá cao nhất hoặc đồng hạng; cả hai cách tăng cường có thể giúp từng check riêng lẻ nhưng chưa có bằng chứng sẽ cải thiện ổn định trên tác vụ mới.

- H1 (subagents so với baseline): Dự đoán `subagents` không cải thiện rõ điểm trung bình trên tác vụ đánh giá. Trên tác vụ học, chỉ `data-learn` gọi subagent (9 lần) nhưng đạt 1/8; `code-learn` và `logs-learn` không giao việc. Mức 3/10 so với 2/10 của `code-learn` không thể quy cho subagent vì `subagent_calls=0`. Chênh lệch token ở `data-learn` cũng không cho biết chi phí giao việc vì baseline bị kẹt trong vòng lặp lệnh sai.
- H2 (skills-auto so với baseline): Dự đoán skill có thể giúp một số check quy ước `rule_` nếu được đọc, nhưng điểm đánh giá tổng thể sẽ không vượt baseline một cách ổn định. Đây là skill do mô hình tự sinh; [SkillsBench v4](https://arxiv.org/abs/2602.12670) ghi nhận skill do người tuyển chọn cải thiện trung bình 16,6 điểm phần trăm, trong khi điều kiện skill tự sinh thấp hơn baseline trên ba cấu hình harness được thử. Kết quả đó là căn cứ để thận trọng, không phải dự báo định lượng cho lab này.
- H3 (tác vụ học so với tác vụ đánh giá): Dự đoán lợi ích của `skills-auto` trên tác vụ học, nếu có, sẽ giảm trên tác vụ đánh giá vì skill được viết từ phản hồi của tác vụ học và tác vụ đánh giá có quy ước mới. [SkillEvolBench](https://arxiv.org/abs/2605.24117) quan sát các cải thiện cục bộ thường không ổn định khi triển khai với skill đã đóng băng; đây là nguy cơ quá khớp cần kiểm tra bằng số liệu của lab.

## 3. Làm quen Deep Agents (Phần 0.3)

1. Tác tử mặc định có 9 công cụ: `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`, `execute` và `task`. Công cụ `execute` cho phép chạy lệnh shell.
2. Dùng để nghiên cứu câu hỏi phức tạp, tìm file/nội dung và thực hiện công việc nhiều bước. Nó có quyền truy cập tất cả công cụ giống tác tử chính.
   Mặc định, mỗi lần gọi là stateless (không giữ trạng thái giữa các lần gọi): subagent chỉ thấy prompt bạn truyền cho nó, không tự thấy lịch sử hội thoại của tác tử chính. Vì vậy, prompt cần chứa đầy đủ ngữ cảnh và yêu cầu kết quả trả về.
3. Từ mô tả task:
   “Launch multiple agents concurrently when their tasks are independent, using a single message with multiple tool calls.”
   → Chạy đồng thời nhiều tác tử khi các nhiệm vụ độc lập.
   Từ mô tả execute:
   “You MUST avoid using search commands like find and grep.”
   → Không dùng execute để tìm kiếm bằng find hoặc grep; dùng công cụ glob và grep riêng.

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

Dữ liệu lấy từ `results/baseline/<tác vụ>/run.json` và `trace.md` của ba tác vụ học. `code-learn` và `data-learn` là các lần chạy lại trong Docker; `logs-learn` giữ kết quả lần chạy trước.

| Tác vụ | Điểm | Token input | Token output | Token total | Giây | `tool_calls` | `subagent_calls` | Trạng thái |
|---|---|---:|---:|---:|---:|---:|---:|---|
| code-learn | 2/10 | 47,020 | 1,137 | 48,157 | 30.0 | 18 | 0 | `error = null`; trace ghi `pytest` không import được `inventory` |
| data-learn | 0/8 | 391,396 | 8,259 | 399,655 | 110.4 | 31 | 0 | `GraphRecursionError`, giới hạn 60 bước; trace lưu được |
| logs-learn | 1/9 | 18,849 | 871 | 19,720 | 12.4 | 3 | 0 | `error = null` |

`data-learn` chạm giới hạn sau 31 lượt gọi công cụ; `trace.md` hiện có 74.205 byte. Trace cho thấy agent lặp lại cùng lệnh `python3 -c` có `with open(...)` sau dấu `;`, gây `SyntaxError`; agent không đổi cách làm và không tạo `answer.json`. Đây là lỗi thực thi/lặp của agent, không phải lỗi Docker. Các check phụ thuộc `answer.json` thất bại vì tệp không tồn tại; không diễn giải chúng thành tám lỗi tính toán độc lập.

**Các check thất bại dùng để phân loại lỗi tác tử:**

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng thực tế |
|---|---|---|---|
| code-learn | `visible_suite_passes` | G. Suite chưa đạt; chưa đủ chi tiết để xác định riêng test lỗi | Checker ghi `1 failed, 5 passed in 0.02s`; trace cho thấy lệnh agent chạy `pytest workspace/tests/` dừng khi import với `ModuleNotFoundError: No module named 'inventory'`. |
| code-learn | `parse_price_all_formats` | D. Bỏ sót định dạng giá | `detail`: `wrong for: ['$1,299.50', '$1,000,000.00']`. Trace sửa dấu ngoặc biểu diễn số âm nhưng không loại dấu phẩy trong giá. |
| code-learn | `other_caller_fixed` | D. Lỗi định dạng giá lan sang hàm gọi | `detail`: `to_csv_row returned '<InvalidOperation>'`. `export.py` gọi `parse_price`; trace không sửa hàm xuất CSV. |
| code-learn | `csv_quoting_follows_docstring` | D. Bỏ sót định dạng CSV | `detail`: `to_csv_row returned 'Desk, large "oak",10.00,2'`. Docstring đã được đọc nhưng trace không có thay đổi xử lý dấu phẩy/dấu nháy của tên hàng. |
| code-learn | `rule_type_hints` | E. Vi phạm quy ước tổ chức | `detail`: `RULE: every public function ... has type annotations on all parameters and on the return value.` |
| code-learn | `rule_regression_tests` | E. Vi phạm quy ước tổ chức | `detail`: `RULE: add tests/test_regressions.py ... (at least 3); the file must pass.` |
| code-learn | `rule_changelog` | E. Vi phạm quy ước tổ chức | `detail`: `RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' ... (at least 3 bullets).` |
| data-learn | `north_q1_revenue`, `north_q1_orders`, `top_region`, `missing_amount_orders`, `duplicate_rows_removed`, `rule_money_in_cents`, `rule_meta_block` | G. Agent lặp lệnh lỗi, không tạo đầu ra | Cả bảy check báo `FileNotFoundError` với `workspace/answer.json`. Trace ghi cùng lệnh `python3 -c` và lỗi cú pháp lặp lại 31 lượt; không có bằng chứng để kết luận riêng từng phép tính sai. |
| data-learn | `rule_clean_csv` | G. Agent lặp lệnh lỗi, không tạo đầu ra | Check yêu cầu `workspace/clean.csv`; trace cho thấy agent bị kẹt trước khi tạo đầu ra. |
| logs-learn | `entry_count` | D. Bỏ sót/bóc tách sai bản ghi log | `detail`: `wrong number of entries (got 10)`. |
| logs-learn | `timestamps_utc` | D. Sai xử lý thời gian/múi giờ | `detail`: `4/25 timestamps match`. Nội dung `write_file` trong trace có `2024-05-01T01:04:08-05:00` ở trường `timestamp_utc`, chưa chuyển sang UTC. |
| logs-learn | `exception_fields` | D. Sai ghép traceback vào bản ghi | `detail`: ``21 wrong `exception` values``. |
| logs-learn | `repeat_counts` | D. Sai xử lý thông báo lặp | `detail`: ``21 wrong `repeat_count` values``. |
| logs-learn | `counts_by_service` | D. Sai tổng hợp dữ liệu | `detail`: `counts_by_service: wrong values`. |
| logs-learn | `rule_service_names` | E. Vi phạm quy ước tổ chức | `detail`: `RULE: service names ... lower-case with '-' replaced by '_' (payment-service -> payment_service).` |
| logs-learn | `rule_sorted_errors` | E. Vi phạm quy ước tổ chức | `detail`: ``RULE: `errors` is sorted by service, then by timestamp_utc, ascending.`` |
| logs-learn | `rule_schema_header` | E. Vi phạm quy ước tổ chức | `detail`: `RULE: the top-level object has "schema_version": 2 and "generated_by": "log-triage".` |

**Các check thất bại ghi nhận nhưng không đưa vào thống kê lỗi tác tử:**

| Tác vụ | Check thất bại | Bằng chứng và lý do loại khỏi thống kê |
|---|---|---|
| code-learn | `tests_not_modified` | `detail`: `the original files in tests/ must not be modified`. Kiểm tra file gốc `tasks/code-learn/workspace/tests/test_report.py` trong repo cho thấy SHA-256 thực tế `efb5e7650d4f03356e8353d209fbcfe81505ce2fd648bd558d5ada6e8b92ff19`; sau chuẩn hóa CRLF thành LF khớp hash checker `79e05f4cc2e62a4f606d210b0b08a2cc21777245bc2f6ad244a126e9a2aee00d`. Như vậy nguồn được sao chép đã có thể gây check thất bại mà không cần agent sửa file; trace cũng không có lệnh sửa file test. Không kết luận agent vi phạm ràng buộc này. |

Trong 23 check đưa vào phân loại, G có **9/23**, D **8/23**, E **6/23**. Tám check của `data-learn` cùng phát sinh từ một vòng lặp lệnh sai cú pháp và thiếu đầu ra; không phải tám nguyên nhân độc lập. D vẫn là nhóm lỗi dữ liệu/định dạng nổi bật ở `code-learn` và `logs-learn`; E là các quy ước tổ chức. `parse_price_all_formats` và `other_caller_fixed` cũng liên hệ cùng lỗi xử lý giá.

`python scripts/check_breakdown.py` đếm trực tiếp các artifact: baseline học đạt **3/18** check kỹ thuật, **0/9** check quy ước; token trung bình **155.844**. Sau khi loại riêng check hash test nghi sai do xuống dòng, phần phân tích còn **2/6** ở `code-learn`, **0/5** ở `data-learn` và **1/6** ở `logs-learn`, tổng **3/17** check kỹ thuật. `data-learn` không tạo đầu ra do vòng lặp `SyntaxError`, nên không có bằng chứng về giá trị tính toán. Trace `logs-learn` chỉ đọc `app.log`, không đọc README và không có bước kiểm chứng sau khi ghi JSON. Ở lần `code-learn` mới, lệnh pytest của agent gặp `ModuleNotFoundError: No module named 'inventory'`; ghi nhận đây là lỗi khi agent chạy lệnh kiểm tra, không phải bằng chứng mọi check kỹ thuật đều hỏng. Chưa có bằng chứng đủ để gán nhóm C hoặc F theo định nghĩa của guide.

Skill có thể hướng dẫn kiểm tra các định dạng giá, quoting CSV, timezone, traceback/repeat count, kiểm chứng output và tránh lệnh Python một dòng có câu lệnh `with`; khi lệnh lỗi, agent cần đổi cách tiếp cận thay vì lặp nguyên lệnh. Đây là biện pháp đề xuất, chưa có số liệu chứng minh skill cải thiện kết quả. Skill không khắc phục được khác biệt CRLF/LF hay lỗi môi trường chạy test.

Nguồn: [code-learn run](../results/baseline/code-learn/run.json), [code-learn trace](../results/baseline/code-learn/trace.md), [data-learn run](../results/baseline/data-learn/run.json), [logs-learn run](../results/baseline/logs-learn/run.json), [logs-learn trace](../results/baseline/logs-learn/trace.md).

## 5. Điều kiện `subagents` (Phần 2.3)

Ba subagent được định nghĩa trong `src/lab/subagents.py`: `explorer` khảo sát tài liệu/code/dữ liệu và báo cáo, `implementer` thực hiện thay đổi và chạy kiểm tra, `reviewer` kiểm tra độc lập. Thiết kế tách khảo sát, thực hiện và kiểm chứng; explorer/reviewer được chỉ dẫn không sửa tệp. Đây là ràng buộc bằng prompt, không phải quyền truy cập được backend cưỡng chế.

| Tác vụ | Điểm | Token input | Token output | Token total | Giây | `tool_calls` | `subagent_calls` | Subagent thực sự được gọi |
|---|---|---:|---:|---:|---:|---:|---:|---|
| code-learn | 3/10 | 44,714 | 1,204 | 45,918 | 18.7 | 16 | 0 | Không có |
| data-learn | 1/8 | 243,921 | 6,300 | 250,221 | 110.2 | 13 | 9 | `implementer`: 9 lần |
| logs-learn | 1/9 | 19,572 | 725 | 20,297 | 8.6 | 3 | 0 | Không có |

Cả ba `run.json` có `error = null`, `skills_read = 0` và `skills_modified = false`. Tuy nhiên, `error = null` chỉ cho biết runner không bị ném lỗi, không bảo đảm mọi tool chạy thành công: `code-learn` có lỗi `asyncio` khi gọi pytest; `data-learn` có lỗi DNS khi cài pandas. Check `tests_not_modified` của `code-learn/subagents` cũng thất bại và chịu cùng vấn đề hash CRLF/LF đã ghi ở mục 4.

`python scripts/check_breakdown.py` ghi nhận điều kiện `subagents` trên tác vụ học đạt **5/18** check kỹ thuật, **0/9** check quy ước; token trung bình **105.478**. Số liệu này bao gồm các lần chạy có lỗi công cụ nên chưa đủ để quy chênh lệch cho việc giao việc.

**Quan sát giao việc:**

- `code-learn`: không có tool `task` trong trace; agent chính tự đọc bốn file Python, gọi `edit_file` và thử chạy pytest hai lần. Việc tự xử lý phù hợp với những bước sửa trực tiếp, nhưng trace không ghi rõ lý do không giao việc nên đây chỉ là giải thích khả dĩ. `explorer` và `reviewer` không được gọi. Lỗi `low_stock_follows_docstring` có bằng chứng rõ: sau khi đổi `<=` thành `<`, agent đổi ngược lại thành `<=` dù docstring yêu cầu “strictly below”; checker ghi `low_stock returned ['b', 'A', 'c']`.
- `logs-learn`: không có tool `task`; agent chính đọc log hai lượt rồi ghi `errors.json`. Có thể agent coi đây là nhiệm vụ xử lý trực tiếp, nhưng chưa có bằng chứng về ý định của nó. Không có bước gọi reviewer hay kiểm chứng output trong trace; kết quả vẫn **1/9**, giống baseline.
- `data-learn`: có đúng **9** tool call `task`, tất cả chọn `subagent_type = implementer`. Không có call tới explorer, reviewer hoặc general-purpose. Các lời giao việc lặp lại cùng mô tả; giữa lần đầu và lần thứ hai, agent chính chạy `pip install pandas` nhưng tool trả `Failed to resolve 'pypi.org'`. Đây là bằng chứng lỗi môi trường, không phải bằng chứng bản thân phép tính của agent sai.

**Thông tin được truyền và việc kiểm chứng:**

Lời giao việc của `data-learn` truyền đường dẫn `workspace/sales.csv`, `workspace/answer.json` và năm tên/chức năng chỉ số cần tính. Tuy nhiên, không truyền nội dung README mà agent chính đã đọc: sentinel `-999` nghĩa là missing, các định dạng ngày `YYYY-MM-DD`/`DD/MM/YYYY`/ISO-8601, chuẩn hóa region và quy tắc ngày chỉ có ngày được hiểu là 00:00 UTC. Mô tả Q1 chỉ ghi ngày bắt đầu/kết thúc, thiếu diễn đạt đầy đủ về giới hạn UTC và không nói rõ phải loại missing amount khỏi doanh thu. Câu “following Acme reporting conventions” không chứa các quy ước cụ thể. Các lần giao lại cũng không bổ sung lỗi DNS hoặc chỉ dẫn dùng thư viện chuẩn thay pandas. Không khẳng định subagent thực tế có đọc README hay không vì trace không chứa hoạt động bên trong subagent.

Một báo cáo của implementer thừa nhận “Chưa thực hiện các phép tính cụ thể ... Cần thực hiện các phép toán để tính toán chính xác”, trong khi vẫn đưa các chỉ số bằng 0. Các báo cáo khác nói thiếu pandas hoặc thiếu dữ liệu; đó là lời báo cáo của subagent, không phải log thực thi bên trong đã được quan sát. Sau chín lần giao việc, agent chính ghi `answer.json` với bốn chỉ số bằng 0 và `top_region = North`, rồi kết luận đã tính toán. Trace không có bước đọc lại answer, chạy kiểm tra hay gọi reviewer sau kết quả cuối. Checker xác nhận `north_q1_revenue`, `north_q1_orders`, `missing_amount_orders`, `duplicate_rows_removed` đều sai với `got 0`; chỉ `top_region` đạt, cả ba check `rule_` thất bại. Có bằng chứng chưa kiểm chứng đủ kết quả được giao, nhưng không gán F chỉ vì phép tính sai: file được báo cáo đã tạo thực sự được ghi.

**So sánh token và thời gian với baseline:**

| Tác vụ | Token baseline | Token subagents | Chênh lệch token | Thời gian baseline → subagents | Diễn giải |
|---|---:|---:|---:|---|---|
| code-learn | 48,157 | 45,918 | -2,239; khoảng 0.95 lần | 30.0 → 18.7 giây | Điểm 2/10 → 3/10; cả hai lần không gọi subagent. Baseline mới ghi lỗi import khi agent chạy pytest. |
| data-learn | 399,655 | 250,221 | -149,434 | 110.4 → 110.2 giây | Baseline lặp lệnh sai cú pháp đến giới hạn; subagents trước đó gặp lỗi DNS. Không dùng chênh lệch token để kết luận multi-agent tiết kiệm chi phí. |
| logs-learn | 19,720 | 20,297 | +577; khoảng 1.03 lần | 12.4 → 8.6 giây | Điểm giữ nguyên 1/9, không giao việc; một lần chạy không đủ giải thích khác biệt thời gian. |

Đây là kết quả tác vụ học, chưa phải kết quả đánh giá sau đóng băng. Token tính cả các lần gọi model bên trong subagent, còn `tool_calls`/`subagent_calls` chỉ tính luồng chính. `trace.md` có thể cắt nội dung mỗi message ở 1,500 ký tự, nên đoạn trace bị cắt không chứng minh model chỉ nhận từng ấy nội dung. Chưa đủ cơ sở kết luận multi-agent tốt hơn: hai tác vụ không giao việc; tác vụ có giao việc chỉ đạt 1/8 và gặp vấn đề môi trường/lặp giao việc.

Nguồn: [code-learn run](../results/subagents/code-learn/run.json), [code-learn trace](../results/subagents/code-learn/trace.md), [data-learn run](../results/subagents/data-learn/run.json), [data-learn trace](../results/subagents/data-learn/trace.md), [logs-learn run](../results/subagents/logs-learn/run.json), [logs-learn trace](../results/subagents/logs-learn/trace.md).

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Curator chạy **3 lần** (lần đầu và 2 lần chạy lại, đúng giới hạn GUIDE). Lần đầu tạo ba skill: xóa `check-file-existence` vì khuyên tạo tệp placeholder khi thiếu dữ liệu; hai skill còn lại hợp lệ về định dạng nhưng cả ba lượt học đều có `skills_read=0`. Lần chạy lại thứ nhất tạo ví dụ riêng cho `sales.csv`/`answer.json` và một kiểm tra kiểu ngày trước bước chuyển đổi; xóa cả ba skill của lượt này vì không đủ tổng quát hoặc có nguy cơ sai. Trước lần chạy lại cuối, siết prompt curator để yêu cầu quy trình tổng quát, không chép code/đường dẫn từ trace và không giả định thư viện tùy chọn. Lần cuối tạo ba skill; xóa `implement-error-handling` vì gợi ý giá trị mặc định khi lỗi có thể che dữ liệu thiếu. **Hai skill cuối cùng được giữ nguyên văn đầu ra curator**, không sửa tay.

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
| ----- | ----------------------------------- | --------------------------------- | ------------------------------------------------- |
| `check-file-existence` | Tổng quát cho tác vụ xử lý dữ liệu có tệp đầu vào | Quy trình kiểm tra tệp đầu vào trước xử lý là đúng; không xử lý được lỗi lặp lệnh hoặc kiểm chứng tệp đầu ra | 10 dòng; `description` bắt đầu `Use when` và nêu lúc cần kiểm tra tệp; cả ba lượt học `skills_read=0` |
| `validate-data-format` | Tổng quát cho dữ liệu có schema | Kiểm tra cột/kiểu trước xử lý là đúng nhưng chưa nêu các quy ước Acme về tiền, CSV, log hoặc test; chỉ dừng khi không khớp | 10 dòng; `description` bắt đầu `Use when` và nêu lúc cần kiểm tra định dạng; cả ba lượt học `skills_read=0` |

Với bộ skill cuối, lần chạy Phần 3.4 đạt `code-learn` **2/10**, **86.883 token**; `data-learn` **0/8**, **489.448 token**, `GraphRecursionError`; `logs-learn` **0/9**, **30.756 token**. Cả ba có `skills_read=0`, nên không quy bất kỳ check đạt được cho việc làm theo skill. Trace `code-learn` bắt đầu bằng `glob` và đọc tệp code, không đọc `skills/`; trace `data-learn` tiếp tục lặp lệnh `python3 -c` sai; trace `logs-learn` chỉ đọc `app.log` rồi dừng mà không ghi output. Bản chạy thử đầu đã lưu riêng ở `results/skills-auto-attempt1`: `code-learn` **4/10**, `data-learn` **0/8**, `logs-learn` **0/9**, cũng đều `skills_read=0`. Chênh lệch `code-learn` giữa hai lượt khi không đọc skill là bằng chứng nhiễu, không phải bằng chứng skill cải thiện.

## 7. Kết quả so sánh (Phần 4.3, 4.4)

> Dán nội dung `report/table.md` và kết quả `python scripts/check_breakdown.py`. Nêu các lần chạy có `error` hoặc `skills_modified = true` (nếu có) và cách xử lý.

```text
(dán bảng ở đây)
```

## 8. Phân tích

> Trả lời từng câu bằng số liệu từ mục 7 và bằng chứng từ vết. Kết quả âm hoặc không có khác biệt vẫn hợp lệ nếu được phân tích tốt.

1. So với `baseline`, điều kiện nào cải thiện điểm tác vụ **học**? Điều kiện nào cải thiện điểm tác vụ **đánh giá**? Có điều kiện nào cải thiện tác vụ học nhưng không cải thiện tác vụ đánh giá? Nếu có, đó là dấu hiệu gì?
2. Tách điểm thành check kỹ thuật và check quy ước (`rule_`). Skill do curator sinh giúp nhóm check nào? Check quy ước **mới** của tác vụ đánh giá có được skill giúp không, và vì sao?
3. Dựa vào vết và `skills_read`, giải thích một check mà skill giúp đạt và một check mà skill không giúp (skill chưa được đọc, đọc nhưng không làm theo, skill thiếu hoặc sai).
4. Chi phí: so sánh số token trung bình giữa các điều kiện. Điều kiện nào có hiệu quả tốt nhất theo điểm trên mỗi token? Đa tác tử có đáng chi phí trong thí nghiệm này không?
5. Có dấu hiệu rò rỉ dữ liệu hoặc quá khớp nào trong skill sinh ra không? Nhóm đã phòng tránh như thế nào?
6. Nhiễu: so sánh điểm tác vụ học của cùng bộ skill ở Phần 3.4 (đã sao lưu) và sau đóng băng. Chênh lệch bao nhiêu? Nó cho biết điều gì về độ tin cậy của các chênh lệch trong bảng ở mục 7?

## 9. Hạn chế và tính hợp lệ

1. Mỗi vai trò chỉ có ba tác vụ thuộc ba họ do bài lab thiết kế. Trung bình từ ba điểm chịu ảnh hưởng lớn của một tác vụ; không suy rộng sang mọi công việc lập trình, dữ liệu hoặc vận hành.
2. Mỗi tổ hợp điều kiện–tác vụ chỉ có một kết quả chính thức. Nhiệt độ 0 không loại bỏ mọi biến động của mô hình hay công cụ; một chênh lệch nhỏ không đủ chứng minh cải tiến ổn định.
3. Các lần chạy học hiện chưa hoàn toàn đồng nhất về môi trường: `code-learn` và `data-learn` baseline mới chạy trong Docker, còn một số kết quả khác được tạo trực tiếp trên Windows. Lỗi pytest, DNS và vòng lặp `GraphRecursionError` có thể chi phối điểm/token; chỉ diễn giải từng trường hợp theo trace, không quy mọi chênh lệch cho subagent hoặc skill.
4. Check `tests_not_modified` của `code-learn` có dấu hiệu sai do khác biệt CRLF/LF dù trace không sửa file test. Điểm thô chứa ít nhất một check có nguy cơ sai; báo cáo tách nó khỏi lỗi hành vi tác tử.
5. Thí nghiệm dùng một cấu hình `openai:gpt-4o-mini`. Kết quả chỉ phản ánh model và harness này; không đủ để kết luận mọi mô hình hoặc hệ thống multi-agent có cùng hiệu quả.

## 10. Kết luận

> Tối đa 5 câu. Chỉ khẳng định điều số liệu hỗ trợ. Nêu một đề xuất cải tiến tiếp theo.

## Phụ lục

- Lệnh đã chạy (theo thứ tự):
- Thử thách mở rộng (nếu có): hướng chọn, kết quả, nhận xét.
- Ghi chú khác:
