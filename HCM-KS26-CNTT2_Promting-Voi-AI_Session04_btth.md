

PHẦN 1: BÁO CÁO THIẾT KẾ GIẢI PHÁP
### Kiến trúc chuỗi Prompt (Prompt Chaining Architecture)
- **Bước 1: Trích xuất & Chuẩn hóa (Extraction & Normalization):** Nhận văn bản hợp đồng thô, sử dụng CoT để nhận diện các điều khoản cốt lõi (bảo mật, thanh toán, phạt vi phạm, chấm dứt) và lọc bỏ nhiễu/dữ liệu dị biệt.
    
- **Bước 2: Phân tích Rủi ro chuyên sâu (Deep Risk Analysis):** Đánh giá mức độ rủi ro pháp lý và tài chính dựa trên các điều khoản đã trích xuất, áp dụng tư duy logic từng bước (CoT).
    
- **Bước 3: Tổng hợp & Định dạng JSON (Action Plan & Formatting):** Tổng hợp kết quả thành báo cáo hành động chuẩn hóa dưới định dạng JSON Schema nghiêm ngặt để tích hợp hệ thống downstream.
### Cơ chế xử lý đứt gãy & Dữ liệu dị biệt (Resilience & Error Handling)

- **Structured Output Enforcement:** Ép buộc LLM trả về cấu trúc định dạng rõ ràng (JSON thuần túy hoặc XML tags) để dễ dàng parsing bằng code.
    
- **Fault Tolerance & Fallback (Chống đứt gãy):** Sử dụng các lớp kiểm tra cú pháp (JSON Schema Validation). Khi LLM trả về dị biệt hoặc markdown bị lỗi, hệ thống tự động kích hoạt Self-Correction Prompt để LLM tự sửa lỗi thay vì crash hệ thống.
    
- **Token Optimization:** Chia nhỏ ngữ cảnh (Context Partitioning). Thay vì gửi toàn bộ tài liệu lớn qua mọi bước, mỗi prompt chỉ nhận trường dữ liệu tối thiểu cần thiết.

Phần 2: chuỗi Prompt thực thi:

### Bước 1: Trích xuất thông tin cốt lõi (Extraction Prompt)

Markdown

```
System: hãy đóng vai là chuyên gia pháp lý và phân tích dữ liệu văn bản. Nhiệm vụ của bạn là trích xuất các điều khoản cốt lõi từ văn bản hợp đồng được cung cấp. 
Nếu văn bản gặp lỗi dị biệt (quá ngắn, không phải hợp đồng, hoặc chứa ký tự vô nghĩa), đưa nó về trường `status: "error"` kèm lý do, không làm sập hệ thống.

Sử dụng chuỗi suy luận (Chain-of-Thought) bên trong thẻ <thought> trước khi đưa ra kết quả JSON:
1. Đọc lướt văn bản để xác định đây có phải hợp đồng không.
2. Tìm kiếm các điều khoản: Thanh toán, Bảo mật (NDA), Chấm dứt hợp đồng, Phạt vi phạm.
3. Trích xuất chính xác trích đoạn điều khoản liên quan.

Đầu ra bắt buộc là JSON thuần túy theo cấu trúc sau:
{
  "status": "success" | "error",
  "error_message": null | "string",
  "extracted_data": {
    "parties": ["string"],
    "payment_terms": "string",
    "confidentiality_clause": "string",
    "termination_clause": "string",
    "penalty_clause": "string"
  }
}
```

### Bước 2: Phân tích rủi ro chuyên sâu (Risk Analysis Prompt)

Markdown

```
System: Hãy đóng vai Trưởng phòng Quản lý Rủi ro Pháp lý. Dựa trên dữ liệu điều khoản hợp đồng đã trích xuất từ bước trước, hãy phân tích rủi ro pháp lý và tài chính.

Suy luận từng bước (Chain-of-Thought):
1. Đánh giá tính cân bằng của điều khoản thanh toán (có rủi ro chiếm dụng vốn không?).
2. Đánh giá tính chặt chẽ của điều khoản bảo mật (có ngoại lệ và thời hạn rõ ràng không?).
3. Đánh giá tính hợp lý của điều khoản phạt vi phạm (có vượt quá giới hạn luật định không?).
4. Xác định mức độ rủi ro tổng thể (Low, Medium, High).

Đầu ra trả về JSON cấu trúc:
{
  "risk_assessment": {
    "overall_risk_level": "Low" | "Medium" | "High",
    "analysis_details": [
      {
        "clause_name": "string",
        "risk_score": 1-10,
        "potential_issue": "string",
        "recommendation": "string"
      }
    ]
  }
}
```

### Bước 3: Tổng hợp hành động & Chuẩn hóa JSON (Final Synthesis Prompt)

Markdown

```
System: Hãy đóng vai hệ thống tích hợp dữ liệu tự động. Nhiệm vụ của bạn là tổng hợp kết quả từ bước trích xuất và bước phân tích rủi ro để tạo ra báo cáo cuối cùng cho hệ thống quản trị doanh nghiệp (ERP).

Yêu cầu:
- Đảm bảo tính nhất quán của dữ liệu.
- Định dạng chuẩn JSON thuần túy (không kèm markdown như ```json nếu API yêu cầu, hoặc bọc gọn gàng).
- Liệt kê các hành động đề xuất cụ thể (Action Items) cho đội ngũ pháp chế.

Cấu trúc JSON đầu ra:
{
  "contract_summary": {
    "parties": [],
    "risk_level": "string",
    "action_required": true/false
  },
  "action_items": [
    {
      "priority": "High" | "Medium" | "Low",
      "task": "string",
      "deadline_days": 0
    }
  ],
  "system_metadata": {
    "processed_at": "timestamp",
    "status": "completed"
  }
}
```

### PHẦN 3: KỊCH BẢN KIỂM THỬ VỚI DỮ LIỆU DỊ BIỆT

- **Tình huống kiểm thử:** Đầu vào là một chuỗi văn bản bị lỗi font chữ nặng hoặc chỉ chứa vài từ vô nghĩa: `"Hợp đồng... ??? ### 123 abc"`.
    
- **Cơ chế xử lý của chuỗi Prompt:**
    
    1. Ở Bước 1, LLM nhận diện không đủ thông tin hợp đồng thông qua phần `<thought>`, chuyển trạng thái `status: "error"`.
        
    2. Code ứng dụng (Python/Node.js) bắt được `status: "error"` từ JSON trả về của Bước 1, tự động ngắt chuỗi (short-circuit), bỏ qua Bước 2 và Bước 3 để tiết kiệm token và thời gian.
        
    3. Hệ thống trả về thông báo lỗi thân thiện cho người dùng: _"Dữ liệu đầu vào không hợp lệ hoặc không đủ cấu trúc hợp đồng để phân tích."_, đảm bảo hệ thống không bị crash.

Kết quả đầu ra sạch lỗi của AI:

JSON

```
{
  "pipeline_execution_result": {
    
    "step_1_extraction": {
      "status": "success",
      "parties": [
        "Công ty  A",
        "Công ty  B"
      ],
      "extracted_clauses": {
        "payment_terms": "Thanh toán 50% trước khi triển khai, 50% còn lại trong vòng 30 ngày.",
        "confidentiality_clause": "Bảo mật toàn bộ thông tin trong suốt hiệu lực và 2 năm sau khi chấm dứt.",
        "penalty_clause": "Phạt chậm thanh toán 0.5%/ngày trên tổng giá trị hóa đơn quá hạn."
      }
    },

    "step_2_risk_analysis": {
      "overall_risk_level": "Medium",
      "analysis_details": [
        {
          "clause_name": "Penalty Clause",
          "risk_score": 8,
          "potential_issue": "Mức phạt 0.5%/ngày có nguy cơ vượt quá giới hạn luật định.",
          "recommendation": "Cần giới hạn mức trần phạt vi phạm."
        },
        {
          "clause_name": "Confidentiality Clause",
          "risk_score": 3,
          "potential_issue": "Chưa có quy định rõ về cơ chế xử lý khi có sự cố rò rỉ.",
          "recommendation": "Bổ sung điều khoản giải quyết tranh chấp dữ liệu."
        }
      ]
    },

    "step_3_action_plan": {
      "contract_summary": {
        "contract_type": "Hợp đồng Cung cấp Dịch vụ Phần mềm (SaaS)",
        "risk_level": "Medium",
        "action_required": true
      },
      "action_items": [
        {
          "priority": "High",
          "task": "Đàm phán lại điều khoản phạt vi phạm hợp đồng, điều chỉnh mức phạt chậm thanh toán về mức hợp pháp.",
          "deadline_days": 3
        },
        {
          "priority": "Medium",
          "task": "Bổ sung điều khoản chi tiết về phương thức giải quyết tranh chấp khi lộ lọt dữ liệu.",
          "deadline_days": 5
        }
      ]
    }

  },
  "system_metadata": {
    "pipeline_version": "v1.2.0",
    "processed_at": "2026-06-06T10:00:00Z",
    "status": "completed",
    "total_tokens_used": 1420
  }
}
```