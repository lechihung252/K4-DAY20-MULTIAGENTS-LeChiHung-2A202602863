# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin sinh viên và cấu hình

- Họ tên: Lê Chí Hùng
- Mã sinh viên: 2A202602863

- Nhà cung cấp và mô hình: `LAB_MODEL=openai:gpt-4o-mini`, `LAB_TEMPERATURE=0`, `--recursion-limit 40` cho mọi lần chạy chính thức (giảm từ mặc định 60 để chặn chi phí khi tác tử lặp, xem mục 4 và Phụ lục).
- Deep Agents 0.7.21, Python 3.14.7, macOS 26.6.2, chạy trực tiếp (không Docker).
- Số lần chạy tác vụ đã dùng / ngân sách: 5 lần chạy, khoảng 1,75 triệu token (3 lần chạy chính thức baseline trên tác vụ học + 2 lần chạy thử `data-learn` với giới hạn 60 bị loại, xem Phụ lục). Ngân sách API hạn chế nên mỗi cấu hình chỉ chạy một lần.
- Commit của tag `freeze`: _(điền sau Phần 4.1)_

## 2. Giả thuyết (commit TRƯỚC tag `freeze`, Phần 4.0)

- H1 (subagents so với baseline): `subagents` **không** cải thiện điểm trung bình trên tác vụ đánh giá (chênh lệch trong khoảng ±1 check mỗi tác vụ) và tốn nhiều token hơn baseline. Căn cứ: lỗi chiếm đa số ở baseline là lỗi quy trình của chính mô hình (lặp lệnh lỗi, không kiểm chứng đầu ra; mục 4), mà subagent dùng cùng mô hình `gpt-4o-mini` nên mắc cùng lỗi; quy ước Acme không có trong đề nên tác tử chính không thể chuyển nó cho subagent; bài viết của Anthropic về hệ thống nghiên cứu đa tác tử ghi nhận chi phí token khoảng 15 lần so với hội thoại thường.
- H2 (skills-auto so với baseline): `skills-auto` **không** khác baseline một cách có hệ thống trên tác vụ đánh giá (chênh lệch trong khoảng nhiễu ±1 check mỗi tác vụ, không cải thiện check quy ước). Căn cứ: (1) ở Phần 3.4 `gpt-4o-mini` không đọc skill nào (`skills_read = 0/3`) dù `SKILLS_NOTE` yêu cầu đọc trước tiên, nên skill không thể tác động; (2) ngay cả khi được đọc, 3 skill chỉ nhắm vào quy ước của họ `code` (CHANGELOG, type hints) và không có skill nào cho lỗi chiếm đa số (lặp lệnh lỗi, không kiểm chứng; mục 4); (3) SkillsBench ghi nhận skill do mô hình tự sinh trung bình không có lợi.
- H3 (tác vụ học so với tác vụ đánh giá): mức cải thiện của `skills-auto` so với baseline trên tác vụ học lớn hơn trên tác vụ đánh giá (quá khớp với quy ước đã thấy). Căn cứ: SkillEvolBench ghi nhận lợi ích trên tác vụ học thường không chuyển sang tác vụ mới; tác vụ đánh giá thêm một quy ước mới mà skill không thể biết trước.

## 3. Làm quen Deep Agents (Phần 0.3)

Nguồn: `python scripts/tour.py` (mô hình giả, không tốn token).

1. Tác tử mặc định có 9 công cụ: công cụ tệp `ls`, `read_file`, `write_file`, `edit_file`, `delete`, `glob`, `grep`; shell `execute`; giao việc cho subagent `task`. Công cụ cho phép chạy lệnh là `execute` (chỉ có khi backend cài đặt `SandboxBackendProtocol`, ở lab này là `LocalShellBackend`).
2. Mô tả của `task` nói `general-purpose` là tác tử "for researching complex questions, searching for files and content, and executing multi-step tasks", nên dùng khi không chắc tìm đúng tệp trong vài lần thử đầu, và "has access to all tools as the main agent". Về ngữ cảnh: "Each invocation is stateless by default: the agent sees only the prompt you give it and returns a single final report". Tức là subagent **không** thấy lịch sử hội thoại, đề bài hay system prompt của tác tử chính; nó chỉ thấy lời giao việc, và báo cáo của nó không hiện cho người dùng ("relay a summary yourself").
3. Câu hướng dẫn hành vi trong mô tả `task`: "Put full detail in the prompt and state exactly what it should return". Câu trong mô tả `execute`: "You MUST avoid using search commands like find and grep. Instead use the grep, glob tools to search. Use read_file rather than cat/head/tail." Đáng chú ý: mô tả `execute` còn khuyên "Use absolute paths and avoid `cd`", mâu thuẫn với quy ước đường dẫn tương đối của `BASE_PROMPT`. Đây là lý do `PATHS_NOTE` nhấn mạnh dạng `workspace/...` cho cả công cụ tệp và shell.

## 4. Đường cơ sở và phân loại lỗi (Phần 2.2)

Kết quả `baseline` trên tác vụ học (`gpt-4o-mini`, giới hạn 40 bước):

| Tác vụ | Điểm | Token | Tool call | Giây | `error` |
|---|---|---|---|---|---|
| `code-learn` | 6/10 | 73.692 | 24 | 41,4 | không |
| `data-learn` | 0/8 | 205.090 | 21 | 112,4 | `GraphRecursionError` (40 bước) |
| `logs-learn` | 0/9 | 569.358 | 20 | 669,9 | `GraphRecursionError` (40 bước) |

21 check thất bại. Các check cùng nguyên nhân được gộp thành một dòng (tên từng check được liệt kê):

| Tác vụ | Check thất bại | Nhóm lỗi (A-G) | Bằng chứng (trích ngắn từ `detail` hoặc vết) |
|---|---|---|---|
| `code-learn` | `csv_quoting_follows_docstring` | A (bỏ qua đặc tả), kèm B | `detail`: "to_csv_row returned 'Desk, large \"oak\",10.00,2'", tức là trường có dấu phẩy và ngoặc kép không được bao trong ngoặc kép như docstring yêu cầu. Vết: tác tử có đọc `export.py` nhưng không sửa tệp này; mọi `edit_file` chỉ nhắm `pricing.py` và `report.py`. Tác tử dừng ngay khi test có sẵn đạt ("6 passed in 0.01s"), dù đề nói "The visible tests do not cover everything: the docstrings are the specification". |
| `code-learn` | `rule_type_hints` | E | `detail`: "RULE: every public function (name not starting with '_') in the package has type annotations on all parameters and on the return value." |
| `code-learn` | `rule_regression_tests` | E | `detail`: "RULE: add tests/test_regressions.py with one test function per bug you fixed (at least 3); the file must pass." |
| `code-learn` | `rule_changelog` | E | `detail`: "RULE: record each fix in CHANGELOG.md under the heading '## Unreleased' as a bullet '- fix(<function name>): ...'". Vết: tác tử không đọc `README.md` hay `CHANGELOG.md` của workspace (lệnh đầu tiên là `glob workspace/inventory/**/*.py`, sau đó chỉ đọc 4 tệp trong `inventory/`). |
| `data-learn` | `north_q1_revenue`, `north_q1_orders`, `top_region`, `missing_amount_orders`, `duplicate_rows_removed`, `rule_money_in_cents`, `rule_meta_block` | G (lặp công cụ, không thoát khỏi lỗi) | `detail`: "FileNotFoundError: ... workspace/answer.json": tệp chưa bao giờ được tạo. Vết: sau khi đọc `sales.csv` và `README.md`, tác tử gọi `execute` **19 lần** với gần như cùng một lệnh `python3 -c "import pandas as pd; ...; with open(...)..."` viết trên một dòng. 18 lần nhận lỗi cú pháp (`File "<string>", line 1`, vì `with`/`for` không viết được sau `;`) và không đổi cách làm (ví dụ ghi script ra tệp) cho đến khi hết 40 bước. Ở lần chạy thử với giới hạn 60, lỗi lặp lại y hệt; lần đó còn có `ModuleNotFoundError: No module named 'pandas'`. |
| `data-learn` | `rule_clean_csv` | G, kèm E | `detail`: "RULE: write workspace/clean.csv with the header order_id,timestamp_utc,region,amount_cents; ...". Quy ước này không có trong đề, nhưng tệp cũng không được tạo vì vòng lặp ở dòng trên. |
| `logs-learn` | `valid_structure`, `entry_count`, `timestamps_utc`, `exception_fields`, `repeat_counts`, `counts_by_service`, `rule_service_names`, `rule_sorted_errors`, `rule_schema_header` | B (không kiểm chứng), kèm D | `detail`: "JSONDecodeError: Expecting ',' delimiter: line 1 column 9504". Vết: **0 lần gọi `execute`**; tác tử đọc `app.log` rồi gọi `write_file` **18 lần**, mỗi lần tự gõ tay toàn bộ JSON (khoảng 10 nghìn ký tự, 48.476 token đầu ra). Nó không viết parser, không chạy `python -m json.tool` để kiểm tra, nên tệp cuối cùng sai cú pháp. Ngay trong bản ghi đầu, `"exception":"null"` là chuỗi chứ không phải `null` (nhóm D). |

Bằng chứng phủ định từ `python scripts/check_breakdown.py`:

```text
condition     role    technical  house rules  mean tokens  read a skill
baseline      learn     6/18         0/9          282,713      0/3
```

Nhận xét:

- **Nhóm chiếm đa số theo số check là G và B** (lỗi quy trình: lặp lệnh lỗi, ghi đầu ra mà không kiểm chứng), với 16/21 check thất bại. Khác với kỳ vọng của GUIDE ("với mô hình mạnh, phần lớn lỗi thuộc nhóm E"), `gpt-4o-mini` thất bại trước khi đến được phần quy ước ở 2/3 tác vụ. Nguyên nhân chung: mô hình không phản ứng với phản hồi lỗi của công cụ (cùng lệnh lỗi gửi lại 19 lần) và không tách "tính toán" ra khỏi "ghi đầu ra" (không viết script, gõ tay JSON).
- **Ở tác vụ duy nhất tác tử hoàn thành (`code-learn`), lỗi chủ yếu là E** (3/4 check thất bại). Check kỹ thuật đạt 6/7 tại đây: nhóm C (vá triệu chứng) không xuất hiện, vì `other_caller_fixed` và `parse_price_all_formats` đều đạt (sửa ở hàm dùng chung `parse_price`). Trên cả 3 tác vụ, check kỹ thuật chỉ đạt 6/18 và check quy ước 0/9. Phần lớn số thất bại kỹ thuật này là hệ quả của G/B (không có tệp hoặc tệp hỏng), chứ không phải do hiểu sai dữ liệu.
- Lỗi `GraphRecursionError` ở `data-learn` và `logs-learn` được tính là lỗi của tác tử chứ không phải lỗi hạ tầng: API trả lời bình thường, và tác tử tự dùng hết ngân sách bước bằng các lệnh lặp.
- **Skill có thể phòng ngừa không?** Có, về nguyên tắc: các quy tắc như "viết script Python ra tệp rồi chạy, không dùng `python -c` nhiều câu lệnh", "nếu một lệnh lỗi 2 lần thì đổi cách", "kiểm tra JSON bằng `json.load` trước khi kết thúc", "đọc README/CHANGELOG của workspace trước khi sửa" đều diễn đạt được thành checklist ngắn. Hạn chế: curator chỉ học từ `detail` của check thất bại. Với `data-learn` và `logs-learn`, `detail` gần như chỉ là `FileNotFoundError`/`JSONDecodeError` (trừ `rule_clean_csv`), nên quy ước Acme của hai họ này hầu như không được phản hồi tới curator, chỉ còn phần vết.

## 5. Điều kiện `subagents` (Phần 2.3)

- Các subagent đã định nghĩa (`src/lab/subagents.py`), vai trò tách biệt theo pha của một tác vụ:
  - `explorer`: chỉ đọc (README, docstring, test, tệp quy ước, mẫu dữ liệu và log), báo cáo sự thật kèm tệp nguồn, không sửa tệp. Lý do: lỗi A và E ở baseline đến từ việc không đọc README/CHANGELOG trước khi sửa.
  - `implementer`: thực hiện thay đổi, sửa nguyên nhân gốc, ưu tiên viết script Python rồi chạy, báo cáo tệp đã đổi và đầu ra lệnh. Lý do: lỗi G/B (gõ tay kết quả, lặp `python -c`).
  - `reviewer`: kiểm tra độc lập ở bước cuối, đối chiếu từng yêu cầu của đề với tệp thật trên đĩa, trả về checklist PASS/FAIL, không sửa. Lý do: lỗi B (không kiểm chứng) và F.
  - `description` của mỗi subagent nêu **khi nào** gọi và **cần gửi gì** (đề bài đầy đủ, quy tắc, đường dẫn), vì subagent chỉ thấy lời giao việc (mục 3, câu 2). `build_agent` nối `PATHS_NOTE` vào `system_prompt` của từng subagent.
- `subagent_calls` ở từng tác vụ và nhận xét: _(chờ chạy `python -m lab.runner --condition subagents --tasks learn --recursion-limit 40`)_
- Thông tin thiếu hoặc thừa khi giao việc: _(chờ kết quả)_
- Ảnh hưởng đến token và thời gian: _(chờ kết quả; baseline học trung bình 282.713 token/tác vụ)_

## 6. Self-evolving: skill do curator sinh (Phần 3)

- Số lần chạy curator, số skill bị xóa và lý do: **2 lần chạy, 0 skill bị xóa tay.**
  - Lần 1: mô hình trả 3 khối (`ensure_valid_imports`, `validate_json_structure`, `document_changes`) nhưng tiêu đề khối dùng dấu gạch dưới, khác với `name:` trong frontmatter. `validate_skill` loại cả 3 ("name differs from the block name"), nên không skill nào được ghi.
  - Sửa prompt của curator (không sửa `validate_skill`): thêm quy tắc "`<name>` chỉ gồm chữ thường, số và gạch ngang, và phải giống hệt nhau ở tiêu đề khối và dòng `name:`".
  - Lần 2 (chạy lại lần 1/2 cho phép): 3 skill hợp lệ được ghi. Giữ nguyên đầu ra, không sửa tay nội dung.
- Đầu vào của curator: 21 check thất bại của baseline trên tác vụ học. Chỉ 4 check có `detail` dạng `RULE:` (3 ở `code-learn`, 1 ở `data-learn`); các check còn lại là `FileNotFoundError` hoặc `JSONDecodeError` (mục 4).

| Skill | Tổng quát hay riêng cho tác vụ học? | Đúng hay sai (nêu chỗ sai nếu có) | Độ dài, `description` và `skills_read` ở Phần 3.4 |
|---|---|---|---|
| `validate-file-structure` | Tổng quát, nhưng **quá chung chung**: "Validate the file format against the expected structure", "Use a linter or validator tool". Không nêu hành động cụ thể như "chạy `python -c 'import json; json.load(open(...))'`" hay "viết parser thành script, không gõ tay JSON", nên không nhắm vào lỗi B/G thật sự của `logs-learn` và `data-learn`. Không có quy tắc nào chống vòng lặp lệnh lỗi (G). | Phần lớn vô hại, nhưng bước 1 "ensure it exists before attempting to read **or write**" **sai** với tệp đầu ra: tệp cần tạo chưa tồn tại, nên quy tắc có thể khiến tác tử dừng hoặc bối rối. Bước 5 "implement error handling to catch and log" không liên quan đến tác vụ. | Thân 5 dòng, không thừa. `description` "Use when creating or modifying files to ensure they meet required formats" rộng, gần như mọi tác vụ đều kích hoạt (tốt cho việc được đọc, nhưng ít định hướng). `skills_read` ở Phần 3.4: **0/3 tác vụ** |
| `enforce-type-annotations` | Tổng quát cho tác vụ Python, rút từ `rule_type_hints` của `code-learn`. Không nêu tên hàm hay tệp của tác vụ. | Đúng ý chính (mọi hàm public có annotation cho tham số và giá trị trả về). Hai điểm yếu: bước 5 "include type hints for all **new** functions" **yếu hơn** quy tắc thật (mọi hàm public, kể cả hàm có sẵn); bước 4 đề nghị `mypy`, mà sandbox không cài, nên có thể tốn lệnh vô ích. Bước 2 (`List`, `Dict`) là kiểu cũ nhưng không sai. | Thân 5 dòng. `description` "Use when defining functions..." **hơi hẹp**: tác vụ sửa lỗi chủ yếu *sửa* hàm có sẵn chứ không *định nghĩa* hàm mới, nên tác tử có thể không đọc. `skills_read` ở Phần 3.4: **0/3 tác vụ** |
| `maintain-changelog` | Quy ước Acme (`CHANGELOG.md`, `## Unreleased`, `- fix(<function name>): ...`) nằm trong giới hạn cho phép theo `05_skill_quality.md`, vì đó chính là quy tắc. Bước 4 "at least three entries" lặp lại con số trong `detail` của `code-learn`, là dấu hiệu **quá khớp** nhẹ với tác vụ học. | Đúng và khớp gần nguyên văn với `detail` của `rule_changelog`. | Thân 5 dòng, mệnh lệnh rõ. `description` "Use when making changes to the codebase..." vừa phải, kích hoạt ở mọi tác vụ sửa mã nhưng không ở tác vụ phân tích dữ liệu hoặc log (đúng phạm vi). `skills_read` ở Phần 3.4: **0/3 tác vụ** |

Nhận xét chung:

- **Thiếu skill cho lỗi chiếm đa số.** Không skill nào nhắm vào nhóm G (lặp cùng lệnh lỗi) và B (không chạy lệnh kiểm chứng): đây là 16/21 check thất bại ở baseline (mục 4). Curator ưu tiên các check có `detail` dạng `RULE:`, vì phản hồi đó cụ thể hơn vết.
- **Thiếu skill cho `rule_regression_tests`** (`code-learn`) và cho quy ước của họ `data` (`rule_money_in_cents`, `rule_meta_block`, `rule_clean_csv`). Nguyên nhân: 2 check đầu có `detail` là `FileNotFoundError` nên curator không thấy quy tắc, còn `rule_clean_csv` thì không được chọn trong giới hạn `max_skills=3`.
- **Không rò rỉ:** `validate_skill` không báo định danh nào của tác vụ đánh giá; nội dung không chứa tên tệp dữ liệu, tên hàm hay con số đáp án của tác vụ học.
- **Quyết định:** giữ cả 3 skill. Không skill nào đủ có hại để xóa (bước 1 của `validate-file-structure` là điểm yếu nhất nhưng mơ hồ hơn là sai hẳn). Không dùng lần chạy lại cuối vì ngân sách API, và vì curator có tính ngẫu nhiên nên lần chạy mới chưa chắc sẽ đưa ra skill cho lỗi G/B.

Kiểm tra việc dùng skill trên tác vụ học (Phần 3.4, `python -m lab.runner --condition skills-auto --tasks learn --recursion-limit 40`, sao lưu ở `results/skills-auto-dev/`):

| Tác vụ | baseline | skills-auto (3.4) | `skills_read` | Token | Tool call | `error` |
|---|---|---|---|---|---|---|
| `code-learn` | 6/10 | 5/10 | 0 | 92.293 | 28 | `GraphRecursionError` |
| `data-learn` | 0/8 | 0/8 | 0 | 247.775 | 20 | `GraphRecursionError` |
| `logs-learn` | 0/9 | 1/9 | 0 | 21.999 | 3 | không |

- **Không skill nào được đọc** dù system prompt có `SKILLS_NOTE` ("As your FIRST action, read the SKILL.md of every skill whose description could apply...") và danh sách tên kèm `description` của 3 skill (test `test_skills_are_loaded_only_when_requested` xác nhận skill có trong system prompt). Vết cả 3 tác vụ không có lệnh `read_file` nào vào `skills/`; hành động đầu tiên lần lượt là `glob workspace/inventory/**/*.py`, `read_file workspace/sales.csv`, `read_file workspace/app.log`, giống hệt baseline. `gpt-4o-mini` bỏ qua chỉ dẫn trong system prompt khi đề bài ở tin nhắn người dùng chỉ định việc cụ thể.
- Vì vậy mọi chênh lệch so với baseline (−1, 0, +1 check) **không thể quy cho skill**; đó là nhiễu của mô hình ở nhiệt độ 0. Ví dụ `code-learn` mất `parse_price_all_formats` ("wrong for: ['(12.00)']") vì tác tử gọi `edit_file` 22 lần (21 lần vào `pricing.py`) và chạm giới hạn bước; `logs-learn` lần này viết JSON hợp lệ ngay trong một `write_file` nên đạt `valid_structure`.
- `skills_modified = false` ở cả 3 lần chạy.
- Hệ quả phụ: vì `logs-learn` (3.4) có tệp hợp lệ, `detail` đã nêu 3 quy ước Acme mà baseline không lộ ra (`rule_service_names`, `rule_sorted_errors`, `rule_schema_header`). Curator không dùng được chúng vì nó chỉ đọc điều kiện `baseline` (Phần 3.2).
