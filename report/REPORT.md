# Báo cáo Lab: Self evolving Agentic

## 1. Thông tin sinh viên và cấu hình

- Họ tên: Lê Chí Hùng
- Mã sinh viên: 2A202602863

- Nhà cung cấp và mô hình: `LAB_MODEL=openai:gpt-4o-mini`, `LAB_TEMPERATURE=0`, `--recursion-limit 40` cho mọi lần chạy chính thức (giảm từ mặc định 60 để chặn chi phí khi tác tử lặp, xem mục 4 và Phụ lục).
- Deep Agents 0.7.21, Python 3.14.7, macOS 26.6.2, chạy trực tiếp (không Docker).
- Số lần chạy tác vụ đã dùng / ngân sách: 8 lần chạy với `gpt-4o-mini`, khoảng 2,12 triệu token (3 baseline + 3 skills-auto-dev trên tác vụ học, cộng 2 lần chạy thử `data-learn` với giới hạn 60 bị loại), cộng 2 lần gọi curator. Thêm 3 lần thử chuyển sang Gemini bị loại (Phụ lục). Credit OpenAI hết sau 8 lần chạy, và quota free của Gemini chỉ có 20 request/ngày, nên **thí nghiệm dừng ở tác vụ học**: không có kết quả `subagents` và không có kết quả tác vụ đánh giá (mục 7, 9).
- Commit của tag `freeze`: `ebccc13` (commit giả thuyết: `c82b9b6`). `python scripts/verify_freeze.py`: "checked 0 runs of skill conditions: OK".

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
- `subagent_calls` ở từng tác vụ và nhận xét: **không có dữ liệu.** Điều kiện `subagents` chưa được chạy vì hết ngân sách API (mục 1). Ở baseline, tác tử chính cũng không giao việc cho subagent mặc định `general-purpose` lần nào (`subagent_calls = 0` ở cả 3 tác vụ học, và cả ở 3 lần chạy skills-auto-dev).
- Thông tin thiếu hoặc thừa khi giao việc: không quan sát được. Dự đoán (H1): vì quy ước Acme không có trong đề, tác tử chính không thể chuyển chúng cho subagent, dù `SUBAGENTS_NOTE` yêu cầu "put ALL the task rules and file paths in the delegation message".
- Ảnh hưởng đến token và thời gian: không đo được. Mốc so sánh: baseline học trung bình 282.713 token/tác vụ; mỗi lần giao việc tạo thêm một vòng gọi mô hình mới với ngữ cảnh riêng, nên chi phí kỳ vọng cao hơn baseline.

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

## 7. Kết quả so sánh (Phần 4.3, 4.4)

`python -m lab.compare > report/table.md`:

```text
| Task | baseline |
|---|---|
| code-learn | 6/10 |
| data-learn | 0/8 |
| logs-learn | 0/9 |
| **Mean score - learning tasks** | 0.20 |
| **Mean score - evaluation tasks** | - |
| **Mean tokens per run** | 282,713 |
| **Runs that read a skill** | 0/3 |
```

`python scripts/check_breakdown.py` (sau tag `freeze`):

```text
condition     role    technical  house rules  mean tokens  read a skill
baseline      learn     6/18         0/9          282,713      0/3
```

Bảng chỉ có cột `baseline` vì `subagents` và `skills-auto` chính thức (sau đóng băng) chưa được chạy. Số liệu của skills-auto ở Phần 3.4 (`results/skills-auto-dev/`, bảng không đọc thư mục này) được tính tay để so sánh:

| Điều kiện (tác vụ học) | Điểm | Check kỹ thuật | Check quy ước | Token trung bình | Đọc skill |
|---|---|---|---|---|---|
| baseline | 6/27 (trung bình 0,20) | 6/18 | 0/9 | 282.713 | 0/3 |
| skills-auto-dev (3.4) | 6/27 (trung bình 0,20) | 6/18 | 0/9 | 120.689 | 0/3 |

Các lần chạy có `error`:
- `GraphRecursionError` (40 bước): baseline `data-learn`, `logs-learn`; skills-auto-dev `code-learn`, `data-learn`. Tất cả do tác tử lặp (mục 4), không phải lỗi hạ tầng, nên vẫn giữ và chấm trên workspace hiện có.
- Không lần chạy nào có `skills_modified = true`.

## 8. Phân tích

1. **Cải thiện tác vụ học và đánh giá.** Trên tác vụ học, không điều kiện nào cải thiện so với baseline: skills-auto-dev đạt cùng tổng 6/27 check (−1 ở `code-learn`, 0 ở `data-learn`, +1 ở `logs-learn`). `subagents` chưa chạy. Không có dữ liệu tác vụ đánh giá, nên không kiểm chứng được H1–H3 hay hiện tượng "cải thiện tác vụ học nhưng không cải thiện tác vụ đánh giá". Kết quả tác vụ học phù hợp với H2 (skill không tạo khác biệt có hệ thống), nhưng chỉ ở tác vụ học.
2. **Check kỹ thuật và check quy ước.** Cả hai điều kiện đạt 6/18 check kỹ thuật và 0/9 check quy ước. Skill không giúp nhóm nào, vì không được đọc (`skills_read = 0/3`). Check quy ước mới của tác vụ đánh giá không đo được. Theo thiết kế, skill cũng không thể giúp ở đó: curator chỉ thấy `detail` của tác vụ học, và cả 3 skill chỉ mô tả quy ước của họ `code`.
3. **Một check skill giúp và một check skill không giúp.** Không có check nào đạt nhờ skill. `valid_structure` của `logs-learn` đạt ở skills-auto-dev nhưng vết không có lệnh đọc `skills/`, và hành động chỉ là `read_file app.log` ×2 rồi một `write_file`, nên đây là nhiễu chứ không phải tác dụng của skill `validate-file-structure`. Check skill lẽ ra giúp nhưng không giúp: `rule_changelog` của `code-learn`. Skill `maintain-changelog` mô tả gần nguyên văn quy tắc ("Create a new entry under the '## Unreleased' section ... '- fix(<function name>): <short description>'"), nhưng tác tử không đọc skill (0 lệnh `read_file` vào `skills/`) và không mở `CHANGELOG.md`. 22 lần `edit_file` của nó chỉ nhắm vào `pricing.py` và `report.py`, nên check vẫn FAIL. Nguyên nhân thuộc loại "skill chưa được đọc", không phải "skill sai".
4. **Chi phí.** Token trung bình: baseline 282.713, skills-auto-dev 120.689. Điểm trên mỗi triệu token: baseline 6/0,848 ≈ 7,1 check; skills-auto-dev 6/0,362 ≈ 16,6 check. Nhưng gần như toàn bộ chênh lệch đến từ một tác vụ: `logs-learn` baseline gõ tay JSON 18 lần (569.358 token), còn ở skills-auto-dev chỉ cần 3 tool call (21.999 token). Vì skill không được đọc, đây là phương sai của việc tác tử có rơi vào vòng lặp hay không, không phải hiệu quả của skill. Chi phí bị chi phối bởi số bước theo dạng gần bình phương: mỗi lần gọi gửi lại toàn bộ lịch sử. Input trung bình mỗi lần gọi ở `logs-learn` baseline là khoảng 26.000 token, so với khoảng 3.000 ở `code-learn`. Đa tác tử không đo được; với một mô hình đã hay lặp, thêm vòng gọi subagent nhiều khả năng chỉ tăng chi phí.
5. **Rò rỉ và quá khớp.** `validate_skill` không phát hiện định danh tác vụ đánh giá nào trong 3 skill. Skill không chứa tên tệp dữ liệu, tên hàm hay con số đáp án. Dấu hiệu quá khớp nhẹ: `maintain-changelog` giữ đúng ngưỡng "at least three entries" lấy từ `detail` của `code-learn`. Biện pháp phòng tránh: curator chỉ đọc `run.json` có `role == "learn"` (kiểm tra bởi `test_04`); không mở `check.py` hay tác vụ đánh giá; giả thuyết được commit trước tag `freeze`. Một lần chạy thử với Gemini đã **tự thoát khỏi sandbox** và đọc `tasks/` của repo, gồm cả `instruction.md` của tác vụ đánh giá (Phụ lục). Lần chạy đó đã bị loại khỏi `results/` (lưu ở `results/_invalid/`, nằm trong `.gitignore`) trước khi curator hoặc bảng so sánh có thể đọc.
6. **Nhiễu.** Không có lần chạy `skills-auto` sau đóng băng để so với Phần 3.4. Các ước lượng nhiễu thay thế từ dữ liệu có sẵn: (a) `data-learn` baseline chạy 3 lần (2 lần giới hạn 60, 1 lần giới hạn 40) đều 0/8 nhưng tốn 399.655, 505.530 và 205.090 token; (b) skills-auto-dev thực chất là baseline có thêm skill không được đọc, nên chênh lệch với baseline (−1, 0, +1 check mỗi tác vụ; token `logs-learn` chênh 26 lần) là thước đo trực tiếp của nhiễu giữa hai lần chạy. Hệ quả: với một lần chạy mỗi cấu hình, chênh lệch dưới khoảng 1 check mỗi tác vụ và chênh lệch token dưới một bậc độ lớn đều không đáng tin cậy.

## 9. Hạn chế và tính hợp lệ

1. **Thí nghiệm chưa hoàn tất.** Không có kết quả `subagents` và không có kết quả tác vụ đánh giá cho điều kiện nào. Vì vậy H1–H3 không được kiểm chứng, và mọi kết luận ở mục 8 chỉ áp dụng cho 3 tác vụ học. Nguyên nhân là ngân sách: credit OpenAI hết sau 8 lần chạy, quota free của Gemini là 20 request/ngày/model, trong khi một lần chạy cần 20–40 request.
2. **Một mô hình yếu, chọn vì ngân sách.** `gpt-4o-mini` kết thúc 4/6 lần chạy chính thức bằng vòng lặp đến giới hạn bước. Điểm phản ánh khả năng thoát vòng lặp nhiều hơn là tác dụng của skill. Kết luận không tổng quát hóa được cho mô hình mạnh hơn, nơi GUIDE kỳ vọng lỗi tập trung ở nhóm E.
3. **Mỗi cấu hình chạy một lần, chỉ 3 tác vụ.** Như mục 8.6 cho thấy, chênh lệch ±1 check và token dao động tới 26 lần giữa hai lần chạy gần như tương đương. Mọi chênh lệch nhỏ trong mục 7 không có ý nghĩa thống kê.
4. **Giới hạn đệ quy 40 cắt ngắn các lần chạy lặp.** Giới hạn này giảm chi phí nhưng làm điểm của tác vụ bị lặp gần như luôn bằng 0, che mất khác biệt nhỏ giữa các điều kiện.
5. **Phản hồi cho curator nghèo đi khi tác tử không tạo đầu ra.** `detail` chỉ nêu quy tắc khi tệp tồn tại. Hai trong ba tác vụ học thất bại vì không có tệp hoặc tệp hỏng, nên curator chỉ thấy 4 quy tắc `RULE:`, thiên lệch về phía kết luận "skill tự sinh không giúp".
6. **Sandbox không cách ly.** Sandbox chỉ là thư mục tạm. Một tác tử (Gemini) đã dùng `pip list` để tìm đường dẫn repo rồi `grep` vào `tasks/`, có thể đọc `check.py` (đáp án) hoặc `.env` (khóa API). Kết quả của một tác tử chủ động như vậy không còn đo năng lực làm tác vụ. Môi trường cũng không có `pandas`, một yếu tố không liên quan tới điều kiện thí nghiệm nhưng ảnh hưởng tới `data-learn`.
7. **Vết chỉ gồm luồng chính** (hạn chế của harness): việc subagent làm bên trong không hiện trong `trace.md`.

## 10. Kết luận

Với `gpt-4o-mini`, tác tử Deep Agents mặc định chỉ đạt 6/27 check trên 3 tác vụ học (0/9 check quy ước). Lỗi chủ yếu là lỗi quy trình: lặp lại lệnh lỗi, và ghi đầu ra mà không kiểm chứng. Curator sinh được 3 skill hợp lệ nhưng chỉ nhắm vào quy ước của họ `code`, còn tác tử không đọc skill nào (`skills_read = 0/3`). Do đó skills-auto không khác baseline trên tác vụ học (6/27 so với 6/27), và các chênh lệch quan sát được nằm trong biên nhiễu giữa hai lần chạy. Vì hết ngân sách, thí nghiệm không có kết quả `subagents` và tác vụ đánh giá, nên H1–H3 chưa được kiểm chứng. Đề xuất tiếp theo: chạy lại trong container chỉ chứa sandbox (không mount repo), với một mô hình mạnh hơn, và cho mỗi cấu hình ít nhất 3 lần chạy để tách tác dụng khỏi nhiễu.

## Phụ lục

- Lệnh đã chạy (theo thứ tự):
  1. `pytest` (32 passed, offline; các tệp `test_01` đến `test_04`)
  2. `python -c "from lab.model import make_model; print(make_model().invoke('Reply with OK').content)"` (OK)
  3. `python -m lab.runner --condition baseline --tasks data-learn` (giới hạn 60): 0/8, 399.655 token, `GraphRecursionError`, `trace.md` rỗng vì cài đặt ban đầu dùng `agent.invoke`. Đã sao lưu ở `results/baseline-attempt1/`.
  4. Đổi `run_task` sang `agent.stream(..., stream_mode="values")` (mở rộng tùy chọn trong `03_runner.md`, mục 8) để vẫn có vết khi lỗi; test vẫn đạt.
  5. `python -m lab.runner --condition baseline --tasks data-learn` (giới hạn 60): 0/8, 505.530 token, 30 tool call. Đã sao lưu ở `results/baseline-limit60/`.
  6. Quyết định dùng `--recursion-limit 40` cho mọi lần chạy chính thức (tiết kiệm token, giữ so sánh công bằng).
  7. `python -m lab.runner --condition baseline --tasks learn --recursion-limit 40`
  8. `python scripts/tour.py`, `python scripts/check_breakdown.py`
  9. `python -m lab.curator` (lần 1): 0 skill hợp lệ (tên khối có dấu gạch dưới); sửa prompt curator.
  10. `python -m lab.curator` (lần 2): 3 skill: `validate-file-structure`, `enforce-type-annotations`, `maintain-changelog`.
  11. `python -m lab.runner --condition skills-auto --tasks learn --recursion-limit 40`, rồi `mv results/skills-auto results/skills-auto-dev`.
  12. Hết credit OpenAI. Thử chuyển sang Gemini (bị loại, xem dưới).
  13. `git commit -m hypotheses` (`c82b9b6`), `git commit --allow-empty -m "freeze skills"` và `git tag freeze` (`ebccc13`), `python scripts/verify_freeze.py` (OK), `python -m lab.compare > report/table.md`, `python scripts/check_breakdown.py`.
- Thử chuyển sang Gemini (mọi kết quả bị loại, không dùng trong báo cáo):
  - `gemini-3.8-flash` qua endpoint tương thích OpenAI: lỗi 400 "Function call is missing a thought_signature" (endpoint này không giữ chữ ký suy nghĩ của Gemini 3 qua các vòng gọi công cụ).
  - `gemini-2.5-flash`: lỗi 404 "no longer available to new users".
  - `google_genai:gemini-3.8-flash` (thư viện `langchain-google-genai`, giữ được chữ ký): baseline `code-learn` chạy 763 giây, 12 tool call, 58.281 token, rồi dừng vì lỗi 429 (quota free 20 request/ngày/model). Trong vết, tác tử **không sửa mã mà dò môi trường**: `glob` toàn sandbox, `git status`, `env`, `pip list` (lộ đường dẫn repo qua gói cài editable), rồi `grep -rn "Acme" ~/<repo>/tasks/`, tức là đọc trực tiếp `instruction.md` của cả tác vụ học và tác vụ đánh giá để tìm quy ước Acme bị ẩn. Đây là hành vi lách bộ chấm (reward hacking) bằng cách thoát khỏi sandbox thư mục tạm. Lần chạy được chuyển sang `results/_invalid/` (trong `.gitignore`); không có khóa API nào xuất hiện trong vết. Biện pháp đề xuất: chạy tác tử trong container chỉ mount sandbox (không mount repo, không mount `.env`), và không cài gói lab ở chế độ editable trong môi trường của tác tử.
- Thử thách mở rộng: không thực hiện đầy đủ. Phát hiện thoát sandbox ở trên là quan sát ngẫu nhiên, liên quan tới hướng 6c nhưng không phải một thí nghiệm có thiết kế.
- Ghi chú khác: các thư mục `results/baseline-attempt1/`, `results/baseline-limit60/`, `results/skills-auto-dev/` không được `lab.compare` đọc; chúng dùng cho Phụ lục và mục 8.6.
