# Database v1 — Bản tin cá nhân hóa

Cập nhật: 13/11/2026. Database chính: PostgreSQL.

Tài liệu này mô tả mô hình dữ liệu logic cho MVP, chưa phải migration SQL. Các mục ghi **đề xuất v1** là lựa chọn thiết kế để hoàn thiện mô hình, chưa phải quyết định sản phẩm đã được xác nhận. Dùng thuật ngữ `newsletter` cho bản tin trong toàn bộ thiết kế này.

## 1. Phạm vi và các quyết định đã thống nhất

- User đăng ký email, chọn topic muốn nhận và cấu hình giờ nhận bản tin.
- User có thể nhận nhiều bản tin trong ngày; mỗi khung giờ có thể có cách đọc riêng.
- Mỗi cấu hình có topic, giới hạn số tin và mức độ chi tiết riêng. Ví dụ sáng đọc lướt, tối đọc sâu.
- Có quản lý topic tập trung; cần hỗ trợ việc thêm topic và xem xét topic do user đề xuất.
- Có cảnh báo tin khẩn cấp liên quan đến topic user quan tâm.
- Kênh gửi MVP là email.
- Cấu hình bản tin và bản tin thực tế đã tạo là hai loại dữ liệu khác nhau. Đổi cấu hình không làm đổi lịch sử.

**Đề xuất v1:** một cấu hình bản tin có một giờ gửi mỗi ngày. User muốn cùng nội dung ở nhiều giờ có thể sao chép cấu hình. Số lần nhận định kỳ mỗi ngày được tính từ số cấu hình đang bật, không lưu thêm một bộ đếm độc lập. Cảnh báo khẩn cấp không tính vào số lần này.

## 2. Quy ước dữ liệu

- Tên bảng và cột dùng `snake_case`; khóa chính `id` dùng UUID, trừ bảng liên kết có khóa chính ghép.
- Thời điểm thực tế dùng `timestamptz`; giờ nhận định kỳ dùng `time without time zone` và múi giờ IANA của user, mặc định `Asia/Ho_Chi_Minh`.
- Các bảng thực thể có `created_at`, `updated_at`, trừ bản ghi lịch sử bất biến chỉ cần `created_at`. Các bảng liên kết có thời điểm tạo khi cần truy vết.
- Trạng thái bên dưới là tập giá trị cho phép, có thể triển khai bằng `text` + `CHECK`. Không cho nhập chuỗi tùy ý.
- Các giới hạn số tin phải là số nguyên dương; điểm đánh giá nếu dùng thang 0–1 phải có `CHECK` tương ứng.
- Dữ liệu quan hệ và trường cần lọc dùng cột/bảng riêng. JSONB chỉ dành cho snapshot, kết quả AI và metadata có cấu trúc biến đổi.
- Email giữ bản gốc để hiển thị và có `email_normalized` để đối chiếu duy nhất. **Đề xuất v1:** trim và chuyển chữ thường; không tự bỏ dấu chấm hoặc phần `+tag` theo quy tắc riêng của nhà cung cấp.
- Không xóa dây chuyền topic, sự kiện hoặc cấu hình đã được dùng trong lịch sử gửi. Dùng trạng thái ngừng hoạt động và khóa ngoại phù hợp.

## 3. Tổng quan quan hệ

```drawio
<mxfile>
  <diagram id="XrpB0DXzddEYBlIb8LkJ" name="Page-1">
    <mxGraphModel dx="1470" dy="1453" grid="1" gridSize="10" guides="1" tooltips="0" connect="1" arrows="1" fold="1" page="1" pageScale="1" pageWidth="850" pageHeight="1100" math="0" shadow="0">
      <root>
        <mxCell id="xoCYBx7sdSMQcaMRAGDy-0" />
        <mxCell id="xoCYBx7sdSMQcaMRAGDy-1" parent="xoCYBx7sdSMQcaMRAGDy-0" />
        <UserObject label="" mermaidData="{&#xa;  &quot;data&quot;: &quot;erDiagram\n    USERS ||--o{ NEWSLETTER_CONFIGS : configures\n    NEWSLETTER_CONFIGS ||--o{ NEWSLETTER_CONFIG_TOPICS : includes\n    TOPICS ||--o{ NEWSLETTER_CONFIG_TOPICS : selected\n    USERS ||--o{ TOPIC_REQUESTS : proposes\n    TOPICS ||--o{ TOPIC_REQUESTS : resolves_to\n    USERS ||--o{ USER_TOPIC_ALERT_SETTINGS : configures\n    TOPICS ||--o{ USER_TOPIC_ALERT_SETTINGS : concerns\n    SOURCES ||--o{ ARTICLES : publishes\n    ARTICLES ||--o{ EVENT_ARTICLES : supports\n    NEWS_EVENTS ||--o{ EVENT_ARTICLES : groups\n    NEWS_EVENTS ||--o{ EVENT_TOPICS : classified_as\n    TOPICS ||--o{ EVENT_TOPICS : relates_to\n    NEWS_EVENTS ||--o{ EVENT_UPDATES : evolves\n    EVENT_UPDATES ||--o{ EVENT_SUMMARIES : summarized\n    EVENT_UPDATES ||--o{ EVENT_ASSESSMENTS : assessed\n    USERS ||--o{ NEWSLETTERS : receives\n    NEWSLETTER_CONFIGS ||--o{ NEWSLETTERS : generates\n    NEWSLETTERS ||--o{ NEWSLETTER_ITEMS : contains\n    EVENT_UPDATES ||--o{ NEWSLETTER_ITEMS : selected\n    EVENT_UPDATES ||--o{ BREAKING_ALERTS : triggers\n    USERS ||--o{ BREAKING_ALERTS : receives\n    NEWSLETTERS ||--o| EMAIL_DELIVERIES : sends\n    BREAKING_ALERTS ||--o| EMAIL_DELIVERIES : sends\n    EMAIL_TEMPLATES ||--o{ EMAIL_DELIVERIES : renders\n    EMAIL_DELIVERIES ||--o{ EMAIL_DELIVERY_ATTEMPTS : retries&quot;,&#xa;  &quot;config&quot;: null&#xa;}" id="sVAeXvakzHPggjMFjTvN-0">
          <mxCell connectable="0" parent="xoCYBx7sdSMQcaMRAGDy-1" style="group;transparentBounds=1;editIcon=1;lockedGroup=0;groupPadding=10;" vertex="1">
            <mxGeometry as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="USERS" mermaidId="n:USERS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="USERS" id="xoCYBx7sdSMQcaMRAGDy-2">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="100" x="327" y="209" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="NEWSLETTER_CONFIGS" mermaidId="n:NEWSLETTER_CONFIGS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="NEWSLETTER_CONFIGS" id="xoCYBx7sdSMQcaMRAGDy-3">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="204" x="746" y="408" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="NEWSLETTER_CONFIG_TOPICS" mermaidId="n:NEWSLETTER_CONFIG_TOPICS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;whiteSpace=wrap;spacingLeft=20;spacingRight=20;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="NEWSLETTER_CONFIG_TOPICS" id="xoCYBx7sdSMQcaMRAGDy-4">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;whiteSpace=wrap;spacingLeft=20;spacingRight=20;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="240" x="1436" y="607" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="TOPICS" mermaidId="n:TOPICS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="TOPICS" id="xoCYBx7sdSMQcaMRAGDy-5">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="100" x="1148" y="209" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="TOPIC_REQUESTS" mermaidId="n:TOPIC_REQUESTS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="TOPIC_REQUESTS" id="xoCYBx7sdSMQcaMRAGDy-6">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="163" x="1035" y="408" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="USER_TOPIC_ALERT_SETTINGS" mermaidId="n:USER_TOPIC_ALERT_SETTINGS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;whiteSpace=wrap;spacingLeft=20;spacingRight=20;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="USER_TOPIC_ALERT_SETTINGS" id="xoCYBx7sdSMQcaMRAGDy-7">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;whiteSpace=wrap;spacingLeft=20;spacingRight=20;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="240" x="426" y="408" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="SOURCES" mermaidId="n:SOURCES" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="SOURCES" id="xoCYBx7sdSMQcaMRAGDy-8">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="104" x="1344" y="10" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="ARTICLES" mermaidId="n:ARTICLES" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="ARTICLES" id="xoCYBx7sdSMQcaMRAGDy-9">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="106" x="1343" y="209" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="EVENT_ARTICLES" mermaidId="n:EVENT_ARTICLES" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="EVENT_ARTICLES" id="xoCYBx7sdSMQcaMRAGDy-10">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="160" x="2119" y="408" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="NEWS_EVENTS" mermaidId="n:NEWS_EVENTS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="NEWS_EVENTS" id="xoCYBx7sdSMQcaMRAGDy-11">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="142" x="1883" y="209" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="EVENT_TOPICS" mermaidId="n:EVENT_TOPICS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="EVENT_TOPICS" id="xoCYBx7sdSMQcaMRAGDy-12">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="144" x="1410" y="408" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="EVENT_UPDATES" mermaidId="n:EVENT_UPDATES" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="EVENT_UPDATES" id="xoCYBx7sdSMQcaMRAGDy-13">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="157" x="1882" y="408" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="EVENT_SUMMARIES" mermaidId="n:EVENT_SUMMARIES" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="EVENT_SUMMARIES" id="xoCYBx7sdSMQcaMRAGDy-14">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="175" x="2473" y="607" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="EVENT_ASSESSMENTS" mermaidId="n:EVENT_ASSESSMENTS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="EVENT_ASSESSMENTS" id="xoCYBx7sdSMQcaMRAGDy-15">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="190" x="1953" y="607" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="NEWSLETTERS" mermaidId="n:NEWSLETTERS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="NEWSLETTERS" id="xoCYBx7sdSMQcaMRAGDy-16">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="141" x="10" y="607" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="NEWSLETTER_ITEMS" mermaidId="n:NEWSLETTER_ITEMS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="NEWSLETTER_ITEMS" id="xoCYBx7sdSMQcaMRAGDy-17">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="183" x="1715" y="806" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="BREAKING_ALERTS" mermaidId="n:BREAKING_ALERTS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="BREAKING_ALERTS" id="xoCYBx7sdSMQcaMRAGDy-18">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="171" x="719" y="607" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="EMAIL_DELIVERIES" mermaidId="n:EMAIL_DELIVERIES" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="EMAIL_DELIVERIES" id="xoCYBx7sdSMQcaMRAGDy-19">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="169" x="871" y="806" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="EMAIL_TEMPLATES" mermaidId="n:EMAIL_TEMPLATES" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="EMAIL_TEMPLATES" id="xoCYBx7sdSMQcaMRAGDy-20">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="170" x="2223" y="607" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="EMAIL_DELIVERY_ATTEMPTS" mermaidId="n:EMAIL_DELIVERY_ATTEMPTS" mermaidBaseStyle="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" mermaidBaseValue="EMAIL_DELIVERY_ATTEMPTS" id="xoCYBx7sdSMQcaMRAGDy-21">
          <mxCell parent="sVAeXvakzHPggjMFjTvN-0" style="shape=table;startSize=80;container=1;collapsible=0;childLayout=tableLayout;fixedRows=1;rowLines=1;fontSize=16;fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;strokeWidth=1;align=center;resizeLast=1;html=1;fillColor=light-dark(#ECECFF,#1f2020);strokeColor=light-dark(#9370DB,#cccccc);fontColor=light-dark(#333333,#cccccc);" vertex="1">
            <mxGeometry height="80" width="237" x="837" y="1005" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="configures" mermaidId="e:USERS-&gt;NEWSLETTER_CONFIGS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.5;" mermaidBaseValue="configures" id="xoCYBx7sdSMQcaMRAGDy-22">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-2" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.5;" target="xoCYBx7sdSMQcaMRAGDy-3">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="587" y="448" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="includes" mermaidId="e:NEWSLETTER_CONFIGS-&gt;NEWSLETTER_CONFIG_TOPICS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" mermaidBaseValue="includes" id="xoCYBx7sdSMQcaMRAGDy-23">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-3" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-4">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1193" y="620" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="selected" mermaidId="e:TOPICS-&gt;NEWSLETTER_CONFIG_TOPICS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.84;exitY=1;entryX=0.16;entryY=0;" mermaidBaseValue="selected" id="xoCYBx7sdSMQcaMRAGDy-24">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-5" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.84;exitY=1;entryX=0.16;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-4">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1474" y="448" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="proposes" mermaidId="e:USERS-&gt;TOPIC_REQUESTS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" mermaidBaseValue="proposes" id="xoCYBx7sdSMQcaMRAGDy-25">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-2" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-6">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="731" y="421" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="resolves_to" mermaidId="e:TOPICS-&gt;TOPIC_REQUESTS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.16;exitY=1;entryX=0.84;entryY=0;" mermaidBaseValue="resolves_to" id="xoCYBx7sdSMQcaMRAGDy-26">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-5" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.16;exitY=1;entryX=0.84;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-6">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1172" y="349" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="configures" mermaidId="e:USERS-&gt;USER_TOPIC_ALERT_SETTINGS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.84;exitY=1;entryX=0.16;entryY=0;" mermaidBaseValue="configures" id="xoCYBx7sdSMQcaMRAGDy-27">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-2" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.84;exitY=1;entryX=0.16;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-7">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="464" y="349" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="concerns" mermaidId="e:TOPICS-&gt;USER_TOPIC_ALERT_SETTINGS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.84;entryX=1;entryY=0.16;" mermaidBaseValue="concerns" id="xoCYBx7sdSMQcaMRAGDy-28">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-5" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.84;entryX=1;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-7">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="907" y="421" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="publishes" mermaidId="e:SOURCES-&gt;ARTICLES#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.5;exitY=1;entryX=0.5;entryY=0;" mermaidBaseValue="publishes" id="xoCYBx7sdSMQcaMRAGDy-29">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-8" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.5;exitY=1;entryX=0.5;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-9">
            <mxGeometry relative="1" as="geometry" />
          </mxCell>
        </UserObject>
        <UserObject label="supports" mermaidId="e:ARTICLES-&gt;EVENT_ARTICLES#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.5;entryX=0;entryY=0.16;" mermaidBaseValue="supports" id="xoCYBx7sdSMQcaMRAGDy-30">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-9" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.5;entryX=0;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-10">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1784" y="249" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="groups" mermaidId="e:NEWS_EVENTS-&gt;EVENT_ARTICLES#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" mermaidBaseValue="groups" id="xoCYBx7sdSMQcaMRAGDy-31">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-11" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-10">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="2072" y="421" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="classified_as" mermaidId="e:NEWS_EVENTS-&gt;EVENT_TOPICS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.84;entryX=1;entryY=0.16;" mermaidBaseValue="classified_as" id="xoCYBx7sdSMQcaMRAGDy-32">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-11" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.84;entryX=1;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-12">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1719" y="421" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="relates_to" mermaidId="e:TOPICS-&gt;EVENT_TOPICS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" mermaidBaseValue="relates_to" id="xoCYBx7sdSMQcaMRAGDy-33">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-5" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-12">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1329" y="421" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="evolves" mermaidId="e:NEWS_EVENTS-&gt;EVENT_UPDATES#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.5;exitY=1;entryX=0.5;entryY=0;" mermaidBaseValue="evolves" id="xoCYBx7sdSMQcaMRAGDy-34">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-11" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.5;exitY=1;entryX=0.5;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-13">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1961" y="349" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="summarized" mermaidId="e:EVENT_UPDATES-&gt;EVENT_SUMMARIES#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.5;" mermaidBaseValue="summarized" id="xoCYBx7sdSMQcaMRAGDy-35">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-13" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.5;" target="xoCYBx7sdSMQcaMRAGDy-14">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="2256" y="647" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="assessed" mermaidId="e:EVENT_UPDATES-&gt;EVENT_ASSESSMENTS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.84;exitY=1;entryX=0.5;entryY=0;" mermaidBaseValue="assessed" id="xoCYBx7sdSMQcaMRAGDy-36">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-13" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.84;exitY=1;entryX=0.5;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-15">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="2048" y="548" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="receives" mermaidId="e:USERS-&gt;NEWSLETTERS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.16;exitY=1;entryX=0.84;entryY=0;" mermaidBaseValue="receives" id="xoCYBx7sdSMQcaMRAGDy-37">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-2" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.16;exitY=1;entryX=0.84;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-16">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="128" y="448" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="generates" mermaidId="e:NEWSLETTER_CONFIGS-&gt;NEWSLETTERS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.84;entryX=1;entryY=0.16;" mermaidBaseValue="generates" id="xoCYBx7sdSMQcaMRAGDy-38">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-3" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.84;entryX=1;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-16">
            <mxGeometry relative="1" x="-0.0021" y="13.24" as="geometry">
              <Array as="points">
                <mxPoint x="449" y="620" />
                <mxPoint x="310" y="590" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="contains" mermaidId="e:NEWSLETTERS-&gt;NEWSLETTER_ITEMS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" mermaidBaseValue="contains" id="xoCYBx7sdSMQcaMRAGDy-39">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-16" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-17">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="933" y="819" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="selected" mermaidId="e:EVENT_UPDATES-&gt;NEWSLETTER_ITEMS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.16;exitY=1;entryX=0.84;entryY=0;" mermaidBaseValue="selected" id="xoCYBx7sdSMQcaMRAGDy-40">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-13" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.16;exitY=1;entryX=0.84;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-17">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1869" y="647" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="triggers" mermaidId="e:EVENT_UPDATES-&gt;BREAKING_ALERTS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.84;entryX=1;entryY=0.16;" mermaidBaseValue="triggers" id="xoCYBx7sdSMQcaMRAGDy-41">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-13" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.84;entryX=1;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-18">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1386" y="620" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="receives" mermaidId="e:USERS-&gt;BREAKING_ALERTS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" mermaidBaseValue="receives" id="xoCYBx7sdSMQcaMRAGDy-42">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-2" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-18">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="573" y="620" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="sends" mermaidId="e:NEWSLETTERS-&gt;EMAIL_DELIVERIES#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToOne;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" mermaidBaseValue="sends" id="xoCYBx7sdSMQcaMRAGDy-43">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-16" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToOne;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=1;exitY=0.84;entryX=0;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-19">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="511" y="819" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="sends" mermaidId="e:BREAKING_ALERTS-&gt;EMAIL_DELIVERIES#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToOne;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.5;exitY=1;entryX=0.16;entryY=0;" mermaidBaseValue="sends" id="xoCYBx7sdSMQcaMRAGDy-44">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-18" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToOne;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.5;exitY=1;entryX=0.16;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-19">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="805" y="747" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="renders" mermaidId="e:EMAIL_TEMPLATES-&gt;EMAIL_DELIVERIES#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.5;entryX=1;entryY=0.16;" mermaidBaseValue="renders" id="xoCYBx7sdSMQcaMRAGDy-45">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-20" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0;exitY=0.5;entryX=1;entryY=0.16;" target="xoCYBx7sdSMQcaMRAGDy-19">
            <mxGeometry relative="1" as="geometry">
              <Array as="points">
                <mxPoint x="1632" y="647" />
              </Array>
            </mxGeometry>
          </mxCell>
        </UserObject>
        <UserObject label="retries" mermaidId="e:EMAIL_DELIVERIES-&gt;EMAIL_DELIVERY_ATTEMPTS#0" mermaidBaseStyle="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.5;exitY=1;entryX=0.5;entryY=0;" mermaidBaseValue="retries" id="xoCYBx7sdSMQcaMRAGDy-46">
          <mxCell edge="1" parent="sVAeXvakzHPggjMFjTvN-0" source="xoCYBx7sdSMQcaMRAGDy-19" style="curved=1;startArrow=ERmandOne;startSize=14;endArrow=ERzeroToMany;endSize=14;strokeColor=light-dark(#333333,#cccccc);strokeWidth=1;html=1;fontSize=14;labelBackgroundColor=light-dark(#F8FFEC4D,#2a2a2a4D);fontFamily=Trebuchet MS,Verdana,Arial,sans-serif;fontColor=light-dark(#333333,#cccccc);exitX=0.5;exitY=1;entryX=0.5;entryY=0;" target="xoCYBx7sdSMQcaMRAGDy-21">
            <mxGeometry relative="1" as="geometry" />
          </mxCell>
        </UserObject>
      </root>
    </mxGraphModel>
  </diagram>
</mxfile>

```

`news_events` biểu diễn sự kiện trong thực tế; `event_updates` biểu diễn diễn biến của sự kiện. Nhiều bài báo nói về cùng một diễn biến không trở thành nhiều mục tin.

## 4. User và quyền nhận email

### `users`

| Cột                               | Ý nghĩa                                                                  |
| --------------------------------- | ------------------------------------------------------------------------ |
| `id`                              | Khóa chính                                                               |
| `email`, `email_normalized`       | Email hiển thị và giá trị đối chiếu; unique trên `email_normalized`      |
| `email_verified_at`               | Thời điểm xác minh; nullable                                             |
| `timezone`                        | Múi giờ IANA                                                             |
| `language`                        | Ngôn ngữ bản tin mặc định, ví dụ `vi`                                    |
| `status`                          | `pending_verification`, `active`, `paused`, `unsubscribed`, `suppressed` |
| `consented_at`, `unsubscribed_at` | Thời điểm đồng ý nhận tin và hủy đăng ký                                 |
| `suppression_reason`              | Lý do chặn gửi, ví dụ bounce hoặc complaint; nullable                    |

Chỉ gửi bản tin/cảnh báo cho user đã xác minh và đủ điều kiện nhận email. `paused` tạm dừng gửi; `suppressed` chặn gửi do trạng thái địa chỉ email. Xác thực email, tạm dừng và hủy đăng ký là các khái niệm riêng.

### `user_access_tokens`

Phục vụ xác minh email, truy cập trang quản lý không mật khẩu và hủy đăng ký nếu chọn luồng dùng token.

Các cột: `id`, `user_id`, `purpose` (`verify_email`, `manage_preferences`, `unsubscribe`), `token_hash` unique, `expires_at`, `used_at`, `revoked_at`, `created_at`.

Chỉ lưu hash của token. Thời hạn và khả năng dùng lại phụ thuộc mục đích; token xác minh/quản lý dùng một lần. Thiết kế endpoint hủy đăng ký cần phù hợp cơ chế email provider, chưa chốt trong tài liệu này. MVP chưa mặc định cần mật khẩu hoặc mạng xã hội đăng nhập.

## 5. Topic và yêu cầu topic

### `topics`

Các cột: `id`, `name`, `slug` unique, `description`, `search_definition` JSONB, `status` (`draft`, `active`, `paused`, `archived`), `origin` (`admin`, `user_request`, `system`), `created_by_user_id` nullable.

- `description` mô tả ranh giới chủ đề, ví dụ AI nói chung hay AI trong y tế Việt Nam.
- `search_definition` là cấu hình thu thập đã chuẩn hóa như từ khóa, thực thể, phạm vi địa lý, ngôn ngữ. Không dùng trực tiếp nội dung user nhập làm chỉ dẫn thực thi.
- Topic được dùng chung giữa các user để tái sử dụng thu thập và xử lý AI.

### `topic_aliases`

Các cột: `id`, `topic_id`, `alias`, `alias_normalized`, `language`. Unique `(topic_id, alias_normalized, language)`.

Ví dụ “AI”, “Artificial Intelligence” và “Trí tuệ nhân tạo” có thể cùng trỏ đến một topic. Alias không bắt buộc duy nhất toàn hệ thống vì từ viết tắt có thể đa nghĩa.

### `topic_requests`

Các cột: `id`, `requested_by_user_id` nullable với đề xuất hệ thống, `origin` (`user`, `system`), `requested_name`, `requested_description`, `status` (`pending`, `needs_clarification`, `approved`, `mapped`, `rejected`), `resolved_topic_id` nullable, `review_note`, `reviewed_at`.

- `approved`: tạo topic mới và liên kết `resolved_topic_id`.
- `mapped`: yêu cầu tương ứng với topic đã có.
- `approved`/`mapped` phải có `resolved_topic_id`; `origin=user` phải có người yêu cầu.
- Yêu cầu topic không tự trở thành topic hoạt động hoặc tự đăng ký vào mọi bản tin.
- Nếu muốn tự thêm topic sau khi duyệt, API phải lưu ngữ cảnh cấu hình đích và kiểm tra lại quyền sở hữu trước khi thêm.

**Chưa chốt:** user nhập topic được kích hoạt tự động sau đối chiếu hay cần admin duyệt. “Upload topic” hiện được hiểu là nhập tên/mô tả; chưa bao gồm upload file hàng loạt.

## 6. Cấu hình bản tin theo khung giờ

### `newsletter_configs`

| Cột                   | Ý nghĩa                                                             |
| --------------------- | ------------------------------------------------------------------- |
| `id`, `user_id`       | Cấu hình thuộc user                                                 |
| `name`                | Ví dụ “Đọc lướt buổi sáng”                                          |
| `local_send_time`     | Giờ gửi theo múi giờ user                                           |
| `detail_level`        | `brief`, `standard`, `detailed`                                     |
| `max_total_stories`   | Số mục tin tối đa trong bản tin                                     |
| `target_read_minutes` | Thời lượng đọc mục tiêu, nullable                                   |
| `status`              | `active`, `paused`, `archived`                                      |
| `revision`            | Tăng khi cấu hình ảnh hưởng nội dung/lịch thay đổi                  |
| `next_run_at`         | Lần chạy kế tiếp, thời điểm tuyệt đối; nullable khi không hoạt động |

**Đề xuất v1:** ngôn ngữ và múi giờ dùng chung ở user; mức chi tiết, giờ nhận và giới hạn tin đặt ở từng cấu hình. Các nhãn giao diện “Đọc lướt”, “Vừa đủ”, “Đọc sâu” ánh xạ sang `detail_level`.

`target_read_minutes` là mục tiêu ước lượng. Khi có cả thời lượng và giới hạn số tin, giới hạn số tin là trần cứng, thời lượng dùng để chọn nội dung phù hợp trong trần đó. Không bảo đảm đồng thời đạt chính xác cả hai.

### `newsletter_config_topics`

Các cột: `newsletter_config_id`, `topic_id`, `max_stories_per_newsletter`, `display_order`. Khóa chính `(newsletter_config_id, topic_id)`.

- Số tin được tính **theo mỗi bản tin**, là giới hạn tối đa. Đây là **đề xuất v1**, vì đơn vị mỗi bản tin hay cả ngày chưa được user chốt riêng.
- Nếu tổng giới hạn topic lớn hơn `max_total_stories`, hệ thống chọn và phân bổ trong giới hạn tổng; không cam kết đủ số tin từng topic.
- Một sự kiện khớp nhiều topic chỉ hiển thị một mục; chọn topic chính để tính hạn mức và lưu các topic liên quan trong snapshot.
- Không đủ tin mới phù hợp thì gửi ít hơn. **Đề xuất v1:** nếu không có tin phù hợp thì ghi `skipped`, không gửi email rỗng.
- Không cần bảng `user_topics` độc lập trong v1: danh sách topic theo dõi được suy ra từ các cấu hình đang hoạt động. Nếu bổ sung bookmark topic độc lập sau này, đó là chức năng khác.

### Quy tắc lập lịch

- Khi đổi múi giờ, giờ nhận, trạng thái hoặc quyền nhận email, tính lại `next_run_at`; các bản tin đã tạo giữ nguyên thời điểm và snapshot.
- Scheduler claim cấu hình đến hạn trong transaction, tạo lần chạy duy nhất rồi cập nhật lịch tiếp theo; không chỉ đọc giờ và gửi trực tiếp.
- Unique `(newsletter_config_id, scheduled_for)` chống tạo trùng một lần chạy. Không đưa `revision` vào khóa này để tránh đổi cấu hình tạo thêm cùng một lần gửi.
- **Đề xuất v1:** giờ không tồn tại do DST được chuyển tới thời điểm hợp lệ kế tiếp; giờ bị lặp chỉ gửi một lần. Phải triển khai bằng thư viện hiểu IANA timezone.
- Nếu hệ thống gián đoạn qua nhiều khung giờ, đề xuất bỏ qua các lần đã quá hạn thay vì gửi dồn; ngưỡng trễ và chính sách khôi phục còn cần chốt.

## 7. Nguồn tin và thu thập

### `sources`

Các cột: `id`, `name`, `base_url`, `feed_url` nullable, `source_type` (`rss`, `api`, `web`), `language`, `publisher_group_key`, `status` (`active`, `paused`, `disabled`), `collection_config` JSONB.

`publisher_group_key` hỗ trợ phân biệt nguồn độc lập với các trang cùng một nhà xuất bản; số lượng URL không phải số lượng xác nhận độc lập. Không lưu API secret trực tiếp trong `collection_config`; chỉ lưu tham chiếu cấu hình secret.

### `topic_sources`

Các cột: `topic_id`, `source_id`. Khóa chính ghép. Biểu diễn nguồn phù hợp/được cấu hình cho topic; không thay thế việc phân loại từng sự kiện.

### `articles`

Các cột: `id`, `source_id`, `original_url`, `canonical_url`, `canonical_url_hash` unique, `title`, `description`, `author` nullable, `language` nullable, `published_at` nullable, `first_seen_at`, `fetched_at` nullable, `content_text` nullable, `content_hash` nullable, `content_status`, `extraction_method` nullable, `extracted_at` nullable, `content_expires_at` nullable, `extraction_error` nullable, `status` (`collected`, `processed`, `failed`, `excluded`).

Giữ URL gốc để user đọc đầy đủ và đối chiếu nguồn; URL chuẩn hóa dùng phát hiện cùng bài. URL không tự chứng minh nội dung chính xác. `content_hash` hỗ trợ phát hiện nội dung sao chép nhưng không unique toàn hệ thống.

V1 lưu tạm văn bản chính đã làm sạch trong `content_text`, không lưu HTML mặc định và chưa cần `content_storage_ref`. Luồng xử lý: lấy metadata → trích xuất từ RSS/API/trang được phép → bỏ menu/quảng cáo → kiểm tra trùng, ngôn ngữ, paywall/nội dung thiếu → gom sự kiện và tóm tắt. Không vượt qua giới hạn truy cập của nguồn. Nội dung web là dữ liệu đầu vào, không phải chỉ dẫn cho AI.

`content_status`: `metadata_only`, `extracted`, `unavailable`, `expired`. Không lấy được toàn văn thì giữ metadata và nguyên nhân, không tạo tóm tắt chi tiết như đã đọc đủ bài. Khi hết hạn lưu và đã xử lý thành công, xóa `content_text`, chuyển sang `expired`; giữ metadata, hash, kết quả và nguồn trích dẫn. `source_references_snapshot` của bản tin lưu tên nguồn, tiêu đề và URL tại lúc tạo. Thời hạn lưu cụ thể chưa chốt.

### `source_collection_states`

Các cột: `source_id` khóa chính, `last_attempt_at`, `last_success_at`, `next_fetch_at`, `cursor` JSONB nullable, `etag` nullable, `last_modified` nullable, `consecutive_failures`, `last_error` nullable.

Bảng lưu tiến độ thu thập, không dùng chung với trạng thái xử lý AI.

## 8. Sự kiện, diễn biến và kết quả AI

### `news_events`

Các cột: `id`, `title`, `status` (`active`, `merged`, `archived`), `merged_into_event_id` nullable, `first_seen_at`, `last_significant_update_at`.

Gom các bài cùng sự kiện để hạn chế trùng lặp. Khi gộp nhầm/correct việc phân nhóm, giữ dấu vết ánh xạ; không sửa mất lịch sử đã gửi. Event trạng thái `merged` phải có đích, không tự trỏ hoặc tạo vòng lặp.

### `event_articles`

Các cột: `event_id`, `article_id`, `relevance_score`. Khóa chính `(event_id, article_id)`; một bài có thể đề cập nhiều sự kiện.

### `event_topics`

Các cột: `event_id`, `topic_id`, `relevance_score`, `classification_version`, `classified_at`. Khóa chính `(event_id, topic_id)`.

### `event_updates`

Các cột: `id`, `event_id`, `version`, `headline`, `facts_snapshot` JSONB, `change_summary`, `is_significant`, `occurred_at` nullable, `detected_at`, `status` (`draft`, `ready`, `retracted`). Unique `(event_id, version)`.

Đây là diễn biến có ý nghĩa, không tạo phiên bản mới chỉ vì thu thập thêm một bài giống nội dung cũ. Bản tin/cảnh báo tham chiếu diễn biến cụ thể để biết chính xác thông tin đã gửi.

### `event_update_sources`

Các cột: `event_update_id`, `article_id`, `evidence_role` (`supports`, `disputes`, `context`). Khóa chính `(event_update_id, article_id)`. Cho phép truy nguồn những bằng chứng được dùng ở từng diễn biến.

### `event_summaries`

Các cột: `id`, `event_update_id`, `language`, `detail_level`, `summary_version`, `headline`, `summary_text`, `context_text` nullable, `why_it_matters` nullable, `model_version`, `prompt_version`, `generated_at`.

Unique `(event_update_id, language, detail_level, summary_version)`. Khi sửa tóm tắt, tạo phiên bản mới thay vì ghi đè bản đã được sử dụng.

Một diễn biến có thể có bản ngắn/vừa/chi tiết, dùng lại giữa nhiều user. Cá nhân hóa v1 tập trung vào chọn tin, độ dài và thứ tự; không bắt buộc gọi AI riêng cho mỗi người.

### `event_assessments`

Các cột: `id`, `event_update_id`, `urgency_score`, `impact_score`, `novelty_score`, `verification_status` (`unverified`, `corroborated`, `verified`, `disputed`), `independent_source_count`, `is_critical`, `rationale`, `evidence_snapshot` JSONB, `model_version`, `policy_version`, `assessed_at`.

Đánh giá là bản ghi theo thời điểm, có thể có nhiều lần cho cùng diễn biến. Lưu tín hiệu AI và kết quả áp dụng chính sách; không xem điểm AI là bằng chứng xác minh. Tiêu chí `verified`, ngưỡng khẩn cấp và độ trễ mục tiêu chưa chốt.

## 9. Cảnh báo tin khẩn cấp

### `user_alert_preferences`

Các cột: `user_id` khóa chính, `enabled`, `max_alerts_per_day`, `min_interval_minutes`, `quiet_hours_start` nullable, `quiet_hours_end` nullable, `critical_bypasses_quiet_hours`.

Giờ yên lặng theo múi giờ user, có thể đi qua nửa đêm; hai đầu cùng null là không có giờ yên lặng. Giá trị giới hạn và quyền bỏ qua giờ yên lặng là **đề xuất cần chốt**, không mặc định bật.

### `user_topic_alert_settings`

Các cột: `user_id`, `topic_id`, `enabled`. Khóa chính ghép. Cấu hình cảnh báo độc lập khung giờ; cùng topic xuất hiện ở ba cấu hình vẫn chỉ phát một cảnh báo cho một diễn biến.

**Đề xuất v1:** cảnh báo chỉ đủ điều kiện khi topic còn nằm trong ít nhất một cấu hình đang hoạt động của user, cấu hình cảnh báo toàn cục và topic đều bật. Xóa topic khỏi tất cả cấu hình sẽ ngừng cảnh báo liên quan, dù bản ghi tùy chọn vẫn được giữ.

### `breaking_alerts`

Các cột: `id`, `user_id`, `event_update_id`, `assessment_id`, `status` (`pending`, `ready`, `cancelled`, `expired`), `matched_topics_snapshot` JSONB, `content_snapshot` JSONB, `policy_snapshot` JSONB, `eligible_at`, `expires_at` nullable, `cancellation_reason` nullable.

Unique `(user_id, event_update_id)`. Một diễn biến khớp nhiều topic chỉ tạo một cảnh báo. Diễn biến tiếp theo chỉ được cảnh báo khi có thay đổi đáng kể và đạt lại các điều kiện; không gửi lại vì có thêm URL.

Đề xuất gửi cảnh báo qua email ngoài lịch sau khi đủ điều kiện. Nếu quá hạn hoặc bị chặn bởi giới hạn/giờ yên lặng, hệ thống phải kiểm tra lại tính thời sự trước khi gửi; không gửi dồn cảnh báo cũ. Chính sách chờ, bỏ qua hay chỉ đưa vào bản tin kế tiếp còn cần chốt.

## 10. Bản tin thực tế và lịch sử nội dung

### `newsletters`

| Cột                                         | Ý nghĩa                                                          |
| ------------------------------------------- | ---------------------------------------------------------------- |
| `id`, `user_id`, `newsletter_config_id`     | Người nhận và cấu hình nguồn                                     |
| `scheduled_for`                             | Thời điểm chạy theo lịch                                         |
| `status`                                    | `pending`, `building`, `ready`, `skipped`, `failed`, `cancelled` |
| `config_revision`, `config_snapshot` JSONB  | Cấu hình, topic, múi giờ và giới hạn tại lúc tạo                 |
| `subject`, `overview`                       | Tiêu đề email và tổng quan, nullable trước khi tạo xong          |
| `estimated_read_minutes`                    | Thời lượng ước lượng thực tế                                     |
| `content_cutoff_at`                         | Mốc chốt dữ liệu đầu vào                                         |
| `generated_at`, `skip_reason`, `last_error` | Metadata tạo bản tin                                             |

Unique `(newsletter_config_id, scheduled_for)`. Các trạng thái này phản ánh tạo nội dung; trạng thái gửi email nằm ở `email_deliveries`.

### `newsletter_items`

Các cột: `id`, `newsletter_id`, `event_update_id`, `event_summary_id`, `primary_topic_id`, `position`, `content_snapshot` JSONB, `source_references_snapshot` JSONB.

Unique `(newsletter_id, event_update_id)` và `(newsletter_id, position)`. Bộ chọn tin còn phải bảo đảm một sự kiện chỉ xuất hiện một lần trong bản tin, kể cả có nhiều diễn biến đủ điều kiện; thông thường chọn diễn biến mới nhất phù hợp.

Snapshot lưu nội dung thực tế, ngôn ngữ, mức chi tiết, topic liên quan và trích dẫn tại thời điểm tạo. Không render lịch sử bằng kết quả AI hiện tại vì nội dung đó có thể thay đổi.

### Quy tắc chọn tin và chống lặp

- Chỉ chọn topic và nội dung phù hợp với cấu hình, trong giới hạn tổng và giới hạn topic.
- Xếp hạng theo độ mới, liên quan, ảnh hưởng và mức đa dạng topic; thuật toán phân bổ cụ thể chưa chốt.
- **Đề xuất v1:** cùng user không nhận lại cùng diễn biến trong nhiều khung giờ. Diễn biến mới đáng kể của sự kiện cũ vẫn có thể được chọn.
- Tin đã gửi cảnh báo không mặc định xuất hiện lại nguyên văn trong bản tin. Có thể đưa lại nếu có diễn biến mới đáng kể hoặc dùng chính sách tổng kết được chốt sau.
- Không dùng trạng thái “đã đọc” để chống lặp. V1 dựa trên lịch sử gửi; provider chấp nhận email không chứng minh user đã đọc.
- Khi tạo song song, cần claim/reservation theo user và diễn biến; xem `user_event_deliveries` bên dưới.

### `user_event_deliveries`

Các cột: `user_id`, `event_update_id`, `email_delivery_id`, `status` (`reserved`, `submitted`, `released`), `reserved_at`, `reservation_expires_at`, `submitted_at` nullable. Khóa chính `(user_id, event_update_id)`.

Đây là sổ chống gửi trùng dùng chung giữa bản tin và cảnh báo. Tạo reservation trong transaction trước khi phát hành; worker khác không chọn diễn biến đã giữ chỗ. `submitted` nghĩa provider đã chấp nhận gửi, không nghĩa email đã vào inbox. Chỉ giải phóng khi chắc chắn chưa gửi; kết quả không xác định phải được đối soát trước.

**Đề xuất v1:** cạnh tranh giữa bản tin và cảnh báo được giải quyết bằng reservation, ưu tiên cảnh báo khi bản tin chưa gửi và còn có thể loại mục đó an toàn. Snapshot bản tin đã phát hành không bị sửa. Chính sách tổng kết lại tin đã cảnh báo sẽ cần mở rộng khóa chống lặp bằng mục đích gửi nếu sau này được bật.

## 11. Template và gửi email

### `email_templates`

Các cột: `id`, `template_key` (`newsletter`, `breaking_alert`, `verify_email`, `manage_preferences`), `language`, `subject_template`, `html_template`, `text_template`, `status` (`active`, `inactive`).

Unique `(template_key, language)`. Template được chỉnh sửa trực tiếp, không quản lý version. Escape nội dung động khi render. Nội dung email sau khi render được lưu tại `email_deliveries`, nên sửa template không làm thay đổi lịch sử email hoặc nội dung retry của delivery đã tạo.

### `email_deliveries`

Các cột: `id`, `user_id`, `newsletter_id` nullable, `breaking_alert_id` nullable, `access_token_id` nullable, `email_template_id`, `recipient_email_snapshot`, `subject_snapshot`, `rendered_html`, `rendered_text`, `idempotency_key` unique, `status` (`queued`, `sending`, `submitted`, `delivered`, `deferred`, `failed`, `bounced`, `complained`, `cancelled`, `unknown`), `provider`, `provider_message_id` nullable, `next_attempt_at`, `submitted_at`, `delivered_at`, `last_error`.

- `CHECK` đúng một trong `newsletter_id`, `breaking_alert_id`, `access_token_id` có giá trị. Unique riêng từng cột tham chiếu không null để mỗi nguồn có một delivery logic.
- Retry dùng lại cùng delivery, không tạo một delivery mới cho cùng email logic.
- Unique `(provider, provider_message_id)` khi ID có giá trị.
- `submitted` là provider chấp nhận; `delivered` là provider báo giao thành công, vẫn không bảo đảm email nằm ở inbox hoặc được đọc.
- Trước lần gửi thực tế, kiểm tra lại quyền nhận email. Hủy đăng ký phải chặn cả bản tin lẫn cảnh báo chưa gửi.
- Khi thay đổi địa chỉ email, phải xác minh lại; không gửi job đang chờ đến địa chỉ snapshot cũ nếu user đã thu hồi quyền nhận tại địa chỉ đó.

### `email_delivery_attempts`

Bảng ghi từng lần hệ thống thử gửi một email qua provider. Một `email_deliveries` là một email cần gửi; có thể có nhiều attempt khi lỗi tạm thời và cần retry. Ví dụ lần 1 provider trả lỗi tạm thời, lần 2 provider chấp nhận: vẫn chỉ một delivery, có hai attempt. Đây không phải lượt user mở email hoặc các lần provider tự thử giao email vào hộp thư.

Các cột: `id`, `email_delivery_id`, `attempt_number`, `started_at`, `finished_at`, `status` (`started`, `accepted`, `failed`, `unknown`), `provider_message_id` nullable, `error_code`, `error_message`. Unique `(email_delivery_id, attempt_number)`.

Timeout sau khi gọi provider có thể là kết quả không xác định. Không retry mù: dùng idempotency của provider nếu có hoặc đối soát. Ràng buộc database không bảo đảm gửi email đúng một lần ở hệ thống ngoài.

### `email_provider_events`

Các cột: `id`, `provider`, `provider_event_id`, `email_delivery_id` nullable trước đối soát, `provider_message_id`, `event_type`, `occurred_at`, `received_at`, `payload` JSONB, `processed_at` nullable. Unique `(provider, provider_event_id)`.

Lưu webhook phục vụ đối soát và chống xử lý lặp. Nếu provider không có event ID, adapter phải tạo khóa khử trùng ổn định. Xử lý webhook đến sai thứ tự, không làm mất trạng thái bounce/complaint vì một sự kiện cũ; suppression user được cập nhật từ sự kiện phù hợp.

## 12. Background jobs

### `background_jobs`

**Đề xuất v1:** dùng bảng job PostgreSQL trước khi chốt broker.

Các cột: `id`, `job_type` (`collect_source`, `process_article`, `assess_event`, `generate_newsletter`, `evaluate_alert`, `send_email`), `payload` JSONB, `idempotency_key` unique, `status` (`pending`, `running`, `succeeded`, `failed`, `cancelled`), `run_after`, `attempt_count`, `max_attempts`, `locked_by`, `locked_until`, `last_error`, `correlation_id`.

- Worker claim bằng transaction và khóa hàng phù hợp, có lease để phục hồi khi process chết.
- Retry có backoff; chỉ claim lại khi lease hết hạn và phải bảo vệ việc worker cũ hoàn tất muộn.
- Tạo bản ghi nghiệp vụ và enqueue job liên quan trong cùng transaction khi cùng database.
- Job lưu ID và tham số xử lý; không sao chép toàn bộ bài báo hoặc email vào payload.
- Nếu chuyển sang broker, cần cơ chế outbox bảo đảm dữ liệu nghiệp vụ và thông điệp đồng bộ. Không mặc định cần thêm Redis/RabbitMQ trong v1.

## 13. Ownership và quyền truy cập

Có thể dùng chung một PostgreSQL instance, chia schema và role:

| Schema         | Thành phần chịu trách nhiệm ghi                                   |
| -------------- | ----------------------------------------------------------------- |
| `core`         | NestJS: user, topic, cấu hình, bản tin, cảnh báo, template, email |
| `collection`   | Java: articles, trạng thái thu thập                               |
| `intelligence` | Python: sự kiện, diễn biến, phân loại, tóm tắt, đánh giá          |
| `jobs`         | Worker tương ứng chỉ claim loại job được cấp quyền                |

`sources` và `topic_sources` thuộc cấu hình nghiệp vụ do NestJS quản lý; Java chỉ đọc và ghi tiến độ vào `source_collection_states`. Python đọc tài liệu được phép và ghi kết quả AI; không sửa user hoặc gửi email. NestJS đọc kết quả AI đã hoàn tất và chịu trách nhiệm phát hành.

Mỗi bảng chỉ có một thành phần sở hữu việc sửa dữ liệu. Migration cần được điều phối chung để quản lý FK xuyên schema. Quyền đọc tối thiểu qua role/view hoặc API nội bộ; không cấp toàn quyền database cho mọi service.

## 14. Index và ràng buộc cần có

Ngoài PK, unique và FK đã nêu:

- Index các cột FK phục vụ truy vấn/xóa kiểm tra; PostgreSQL không tự tạo index cho mọi FK.
- `newsletter_configs(next_run_at)` với điều kiện `status='active'`.
- `newsletters(user_id, scheduled_for DESC)` cho lịch sử bản tin.
- `newsletter_items(event_update_id)` để truy ngược nội dung đã dùng.
- `articles(source_id, published_at DESC)` và index `content_hash` không unique.
- `event_topics(topic_id, relevance_score DESC)` và `event_updates(event_id, detected_at DESC)`.
- `event_assessments(event_update_id, assessed_at DESC)` để lấy đánh giá mới nhất.
- `breaking_alerts(user_id, created_at DESC)` và `user_event_deliveries(user_id, submitted_at DESC)`.
- `email_deliveries(next_attempt_at)` cho trạng thái chờ/thử lại.
- `background_jobs(run_after)` cho `pending`, `background_jobs(locked_until)` cho `running`.
- `CHECK` cho giới hạn số tin, điểm đánh giá, bộ đôi giờ yên lặng và các quan hệ nullable phụ thuộc trạng thái.
- Kiểm tra sở hữu xuyên bảng: cấu hình của bản tin phải thuộc đúng user; delivery và nội dung phải cùng người nhận; summary/assessment phải thuộc đúng diễn biến được tham chiếu. Dùng composite FK khi phù hợp và transaction validation cho quy tắc không biểu diễn được bằng FK đơn.

Không thêm GIN cho mọi JSONB hoặc partition ngay từ đầu; chỉ bổ sung khi có truy vấn và quy mô thực tế cần đến.

## 15. Luồng dữ liệu mẫu

1. User đăng ký email, xác minh, chọn múi giờ và ngôn ngữ.
2. Tạo “Buổi sáng” lúc 8h, đọc lướt, tối đa 5 tin; chọn AI tối đa 3, kinh tế tối đa 3. Tổng thực tế không vượt 5.
3. Tạo “Buổi tối” lúc 20h, đọc sâu, tối đa 10 tin; cấu hình topic riêng.
4. Java thu thập bài; Python gom sự kiện, tạo diễn biến, phân loại và tóm tắt các mức cần dùng.
5. Đến lịch, tạo `newsletters`, chọn tin và giữ chỗ chống trùng, lưu `newsletter_items` cùng snapshot.
6. Render template đang hoạt động, lưu nội dung email đã render, tạo delivery và job gửi. Khi provider chấp nhận, cập nhật delivery và sổ diễn biến đã gửi.
7. Nếu phát hiện diễn biến khẩn cấp phù hợp, đánh giá điều kiện user, tạo `breaking_alerts` và đi qua cùng luồng delivery/chống trùng.
8. Webhook cập nhật kết quả giao email; bounce/complaint có thể chặn các lần gửi sau.

## 16. Lưu trữ, bảo mật và vận hành

- Truy vấn cấu hình/lịch sử luôn kiểm tra user sở hữu, không dựa riêng vào ID do client cung cấp.
- Không log token, secret hoặc toàn bộ payload chứa email không cần thiết; nội dung lỗi cần được lọc dữ liệu nhạy cảm.
- Giới hạn quyền đọc email, nội dung đã render và payload provider. Có backup và kiểm tra khôi phục trước khi vận hành thật.
- Bản tin đã phát hành giữ snapshot bất biến trong thời hạn lưu; thao tác xóa dữ liệu cá nhân phải xử lý cả snapshot, email render và webhook có PII, không chỉ bảng `users`.
- Nội dung bài thu thập, lịch sử job, token hết hạn, webhook và nội dung email có thời hạn lưu riêng. Thời hạn cụ thể chưa chốt; không mặc định giữ vô hạn.
- Theo dõi độ trễ collection → đánh giá → gửi, lỗi AI, hàng đợi quá hạn, bounce, complaint và số cảnh báo bị giới hạn.

## 17. Các điểm cần chốt trước migration

1. Topic user đề xuất: tự kích hoạt sau đối chiếu hay admin duyệt; thêm topic tự động vào cấu hình nào?
2. Xác nhận số tin/topic tính theo mỗi bản tin như đề xuất, hay tổng cả ngày.
3. Giới hạn số cấu hình, topic, tin mỗi bản tin và độ dài chi tiết cho MVP.
4. Chính sách phân bổ khi tổng hạn mức topic vượt giới hạn bản tin; cách xử lý bản tin không có tin mới.
5. Ngưỡng xác minh/khẩn cấp, độ trễ mục tiêu, giới hạn cảnh báo và giờ yên lặng; có gửi ngoài lịch như đề xuất không?
6. Có cần nhắc lại tin đã cảnh báo trong bản tin tổng kết, hay chỉ khi có diễn biến mới?
7. Chính sách lịch quá hạn khi hệ thống gián đoạn và trường hợp user tạo nhiều cấu hình cùng giờ.
8. Luồng truy cập quản lý email, email provider và cách đối soát kết quả gửi không xác định.
9. Thời hạn lưu dữ liệu, chính sách xóa dữ liệu và phạm vi nguồn tin.

Các lựa chọn còn mở được tập trung tại đây để tránh biến giả định thiết kế thành yêu cầu đã thống nhất. Tài liệu sản phẩm/kiến trúc cũ có thể vẫn dùng thuật ngữ hoặc phạm vi trước cuộc thảo luận này; khi triển khai cần đồng bộ chúng với các quyết định được xác nhận tiếp theo.

## 18. Từ điển thuộc tính chuẩn hóa

Phần này là danh sách thuộc tính chuẩn để triển khai; bổ sung kiểu, nullable và mặc định cho mô tả nghiệp vụ ở các phần trên. Mặc định kỹ thuật là đề xuất v1; giới hạn sản phẩm chưa chốt không được tự gán số.

### Quy ước đọc bảng

- `Bắt buộc = Có` tương ứng `NOT NULL`; `Không` cho phép SQL `NULL`, không dùng chuỗi rỗng thay thế.
- `—` ở mặc định nghĩa phải truyền giá trị nếu trường bắt buộc, không phải tự nhận chuỗi rỗng hoặc 0.
- Mọi bảng có `created_at timestamptz NOT NULL DEFAULT now()`. Bảng nhóm **M** có thêm `updated_at timestamptz NOT NULL DEFAULT now()` và cập nhật qua ứng dụng/trigger khi sửa; nhóm **H** chỉ có `created_at`. Hai trường chung này được khai báo tại đây để không lặp trong từng bảng.
- UUID `id` đơn được sinh tự động (`gen_random_uuid()`); ID trong khóa ghép/khóa ngoại lấy từ bản ghi cha. Không đổi khóa chính sau khi tạo.
- `PK ghép` nghĩa tập hợp tất cả cột đánh dấu PK của bảng tạo thành một khóa chính.
- FK mặc định `ON DELETE RESTRICT`, `ON UPDATE RESTRICT`; không cascade xóa lịch sử. Xóa dữ liệu cá nhân dùng workflow chủ động theo phần 16.
- `text` phải có giới hạn chiều dài ở API theo loại dữ liệu; tên/URL/nội dung bắt buộc không được chỉ chứa khoảng trắng. URL bài nguồn là HTTP(S) công khai; crawler kiểm tra đích và redirect trước khi truy cập.
- Hash dùng SHA-256 dạng hex chữ thường 64 ký tự và kiểm tra định dạng; token không lưu bản rõ.
- Các tập trạng thái và unique ghép giữ nguyên theo phần mô tả từng entity. Các giá trị JSONB mặc định rỗng chỉ phục vụ giai đoạn tạo, phải kiểm tra đủ dữ liệu trước `ready`/gửi.

### `users` — nhóm M

| Thuộc tính           | Kiểu PostgreSQL | Bắt buộc | Mặc định               | Khóa / tham chiếu |
| -------------------- | --------------- | -------- | ---------------------- | ----------------- |
| `id`                 | `uuid`          | Có       | `gen_random_uuid()`    | PK                |
| `email`              | `text`          | Có       | —                      | —                 |
| `email_normalized`   | `text`          | Có       | —                      | UNIQUE            |
| `email_verified_at`  | `timestamptz`   | Không    | —                      | —                 |
| `timezone`           | `text`          | Có       | `Asia/Ho_Chi_Minh`     | —                 |
| `language`           | `text`          | Có       | `vi`                   | —                 |
| `status`             | `text`          | Có       | `pending_verification` | —                 |
| `consented_at`       | `timestamptz`   | Không    | —                      | —                 |
| `unsubscribed_at`    | `timestamptz`   | Không    | —                      | —                 |
| `suppression_reason` | `text`          | Không    | —                      | —                 |

### `user_access_tokens` — nhóm H

| Thuộc tính   | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| ------------ | --------------- | -------- | ------------------- | ----------------- |
| `id`         | `uuid`          | Có       | `gen_random_uuid()` | PK                |
| `user_id`    | `uuid`          | Có       | —                   | FK → users.id     |
| `purpose`    | `text`          | Có       | —                   | —                 |
| `token_hash` | `text`          | Có       | —                   | UNIQUE            |
| `expires_at` | `timestamptz`   | Có       | —                   | —                 |
| `used_at`    | `timestamptz`   | Không    | —                   | —                 |
| `revoked_at` | `timestamptz`   | Không    | —                   | —                 |

### `topics` — nhóm M

| Thuộc tính           | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| -------------------- | --------------- | -------- | ------------------- | ----------------- |
| `id`                 | `uuid`          | Có       | `gen_random_uuid()` | PK                |
| `name`               | `text`          | Có       | —                   | —                 |
| `slug`               | `text`          | Có       | —                   | UNIQUE            |
| `description`        | `text`          | Có       | —                   | —                 |
| `search_definition`  | `jsonb`         | Có       | `{}`                | —                 |
| `status`             | `text`          | Có       | `draft`             | —                 |
| `origin`             | `text`          | Có       | —                   | —                 |
| `created_by_user_id` | `uuid`          | Không    | —                   | FK → users.id     |

### `topic_aliases` — nhóm M

| Thuộc tính         | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| ------------------ | --------------- | -------- | ------------------- | ----------------- |
| `id`               | `uuid`          | Có       | `gen_random_uuid()` | PK                |
| `topic_id`         | `uuid`          | Có       | —                   | FK → topics.id    |
| `alias`            | `text`          | Có       | —                   | —                 |
| `alias_normalized` | `text`          | Có       | —                   | —                 |
| `language`         | `text`          | Có       | —                   | —                 |

### `topic_requests` — nhóm M

| Thuộc tính              | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| ----------------------- | --------------- | -------- | ------------------- | ----------------- |
| `id`                    | `uuid`          | Có       | `gen_random_uuid()` | PK                |
| `requested_by_user_id`  | `uuid`          | Không    | —                   | FK → users.id     |
| `origin`                | `text`          | Có       | —                   | —                 |
| `requested_name`        | `text`          | Có       | —                   | —                 |
| `requested_description` | `text`          | Không    | —                   | —                 |
| `status`                | `text`          | Có       | `pending`           | —                 |
| `resolved_topic_id`     | `uuid`          | Không    | —                   | FK → topics.id    |
| `review_note`           | `text`          | Không    | —                   | —                 |
| `reviewed_at`           | `timestamptz`   | Không    | —                   | —                 |

### `newsletter_configs` — nhóm M

| Thuộc tính            | Kiểu PostgreSQL          | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| --------------------- | ------------------------ | -------- | ------------------- | ----------------- |
| `id`                  | `uuid`                   | Có       | `gen_random_uuid()` | PK                |
| `user_id`             | `uuid`                   | Có       | —                   | FK → users.id     |
| `name`                | `text`                   | Có       | —                   | —                 |
| `local_send_time`     | `time without time zone` | Có       | —                   | —                 |
| `detail_level`        | `text`                   | Có       | `standard`          | —                 |
| `max_total_stories`   | `integer`                | Có       | —                   | —                 |
| `target_read_minutes` | `integer`                | Không    | —                   | —                 |
| `status`              | `text`                   | Có       | `paused`            | —                 |
| `revision`            | `integer`                | Có       | `1`                 | —                 |
| `next_run_at`         | `timestamptz`            | Không    | —                   | —                 |

### `newsletter_config_topics` — nhóm M

| Thuộc tính                   | Kiểu PostgreSQL | Bắt buộc | Mặc định | Khóa / tham chiếu              |
| ---------------------------- | --------------- | -------- | -------- | ------------------------------ |
| `newsletter_config_id`       | `uuid`          | Có       | —        | PK; FK → newsletter_configs.id |
| `topic_id`                   | `uuid`          | Có       | —        | PK; FK → topics.id             |
| `max_stories_per_newsletter` | `integer`       | Có       | —        | —                              |
| `display_order`              | `integer`       | Có       | `0`      | —                              |

### `sources` — nhóm M

| Thuộc tính            | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| --------------------- | --------------- | -------- | ------------------- | ----------------- |
| `id`                  | `uuid`          | Có       | `gen_random_uuid()` | PK                |
| `name`                | `text`          | Có       | —                   | —                 |
| `base_url`            | `text`          | Có       | —                   | —                 |
| `feed_url`            | `text`          | Không    | —                   | —                 |
| `source_type`         | `text`          | Có       | —                   | —                 |
| `language`            | `text`          | Không    | —                   | —                 |
| `publisher_group_key` | `text`          | Không    | —                   | —                 |
| `status`              | `text`          | Có       | `paused`            | —                 |
| `collection_config`   | `jsonb`         | Có       | `{}`                | —                 |

### `topic_sources` — nhóm H

| Thuộc tính  | Kiểu PostgreSQL | Bắt buộc | Mặc định | Khóa / tham chiếu   |
| ----------- | --------------- | -------- | -------- | ------------------- |
| `topic_id`  | `uuid`          | Có       | —        | PK; FK → topics.id  |
| `source_id` | `uuid`          | Có       | —        | PK; FK → sources.id |

### `articles` — nhóm M

| Thuộc tính           | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| -------------------- | --------------- | -------- | ------------------- | ----------------- |
| `id`                 | `uuid`          | Có       | `gen_random_uuid()` | PK                |
| `source_id`          | `uuid`          | Có       | —                   | FK → sources.id   |
| `original_url`       | `text`          | Có       | —                   | —                 |
| `canonical_url`      | `text`          | Có       | —                   | —                 |
| `canonical_url_hash` | `text`          | Có       | —                   | UNIQUE            |
| `title`              | `text`          | Có       | —                   | —                 |
| `description`        | `text`          | Không    | —                   | —                 |
| `author`             | `text`          | Không    | —                   | —                 |
| `language`           | `text`          | Không    | —                   | —                 |
| `published_at`       | `timestamptz`   | Không    | —                   | —                 |
| `first_seen_at`      | `timestamptz`   | Có       | `now()`             | —                 |
| `fetched_at`         | `timestamptz`   | Không    | —                   | —                 |
| `content_text`       | `text`          | Không    | —                   | —                 |
| `content_hash`       | `text`          | Không    | —                   | —                 |
| `content_status`     | `text`          | Có       | `metadata_only`     | —                 |
| `extraction_method`  | `text`          | Không    | —                   | —                 |
| `extracted_at`       | `timestamptz`   | Không    | —                   | —                 |
| `content_expires_at` | `timestamptz`   | Không    | —                   | —                 |
| `extraction_error`   | `text`          | Không    | —                   | —                 |
| `status`             | `text`          | Có       | `collected`         | —                 |

### `source_collection_states` — nhóm M

| Thuộc tính             | Kiểu PostgreSQL | Bắt buộc | Mặc định | Khóa / tham chiếu   |
| ---------------------- | --------------- | -------- | -------- | ------------------- |
| `source_id`            | `uuid`          | Có       | —        | PK; FK → sources.id |
| `last_attempt_at`      | `timestamptz`   | Không    | —        | —                   |
| `last_success_at`      | `timestamptz`   | Không    | —        | —                   |
| `next_fetch_at`        | `timestamptz`   | Không    | —        | —                   |
| `cursor`               | `jsonb`         | Không    | —        | —                   |
| `etag`                 | `text`          | Không    | —        | —                   |
| `last_modified`        | `text`          | Không    | —        | —                   |
| `consecutive_failures` | `integer`       | Có       | `0`      | —                   |
| `last_error`           | `text`          | Không    | —        | —                   |

### `news_events` — nhóm M

| Thuộc tính                   | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu   |
| ---------------------------- | --------------- | -------- | ------------------- | ------------------- |
| `id`                         | `uuid`          | Có       | `gen_random_uuid()` | PK                  |
| `title`                      | `text`          | Có       | —                   | —                   |
| `status`                     | `text`          | Có       | `active`            | —                   |
| `merged_into_event_id`       | `uuid`          | Không    | —                   | FK → news_events.id |
| `first_seen_at`              | `timestamptz`   | Có       | `now()`             | —                   |
| `last_significant_update_at` | `timestamptz`   | Không    | —                   | —                   |

### `event_articles` — nhóm M

| Thuộc tính        | Kiểu PostgreSQL | Bắt buộc | Mặc định | Khóa / tham chiếu       |
| ----------------- | --------------- | -------- | -------- | ----------------------- |
| `event_id`        | `uuid`          | Có       | —        | PK; FK → news_events.id |
| `article_id`      | `uuid`          | Có       | —        | PK; FK → articles.id    |
| `relevance_score` | `numeric`       | Có       | —        | —                       |

### `event_topics` — nhóm M

| Thuộc tính               | Kiểu PostgreSQL | Bắt buộc | Mặc định | Khóa / tham chiếu       |
| ------------------------ | --------------- | -------- | -------- | ----------------------- |
| `event_id`               | `uuid`          | Có       | —        | PK; FK → news_events.id |
| `topic_id`               | `uuid`          | Có       | —        | PK; FK → topics.id      |
| `relevance_score`        | `numeric`       | Có       | —        | —                       |
| `classification_version` | `text`          | Có       | —        | —                       |
| `classified_at`          | `timestamptz`   | Có       | `now()`  | —                       |

### `event_updates` — nhóm M

| Thuộc tính       | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu   |
| ---------------- | --------------- | -------- | ------------------- | ------------------- |
| `id`             | `uuid`          | Có       | `gen_random_uuid()` | PK                  |
| `event_id`       | `uuid`          | Có       | —                   | FK → news_events.id |
| `version`        | `integer`       | Có       | —                   | —                   |
| `headline`       | `text`          | Có       | —                   | —                   |
| `facts_snapshot` | `jsonb`         | Có       | `{}`                | —                   |
| `change_summary` | `text`          | Có       | —                   | —                   |
| `is_significant` | `boolean`       | Có       | `false`             | —                   |
| `occurred_at`    | `timestamptz`   | Không    | —                   | —                   |
| `detected_at`    | `timestamptz`   | Có       | `now()`             | —                   |
| `status`         | `text`          | Có       | `draft`             | —                   |

### `event_update_sources` — nhóm H

| Thuộc tính        | Kiểu PostgreSQL | Bắt buộc | Mặc định | Khóa / tham chiếu         |
| ----------------- | --------------- | -------- | -------- | ------------------------- |
| `event_update_id` | `uuid`          | Có       | —        | PK; FK → event_updates.id |
| `article_id`      | `uuid`          | Có       | —        | PK; FK → articles.id      |
| `evidence_role`   | `text`          | Có       | —        | —                         |

### `event_summaries` — nhóm H

| Thuộc tính        | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu     |
| ----------------- | --------------- | -------- | ------------------- | --------------------- |
| `id`              | `uuid`          | Có       | `gen_random_uuid()` | PK                    |
| `event_update_id` | `uuid`          | Có       | —                   | FK → event_updates.id |
| `language`        | `text`          | Có       | —                   | —                     |
| `detail_level`    | `text`          | Có       | —                   | —                     |
| `summary_version` | `integer`       | Có       | —                   | —                     |
| `headline`        | `text`          | Có       | —                   | —                     |
| `summary_text`    | `text`          | Có       | —                   | —                     |
| `context_text`    | `text`          | Không    | —                   | —                     |
| `why_it_matters`  | `text`          | Không    | —                   | —                     |
| `model_version`   | `text`          | Có       | —                   | —                     |
| `prompt_version`  | `text`          | Có       | —                   | —                     |
| `generated_at`    | `timestamptz`   | Có       | `now()`             | —                     |

### `event_assessments` — nhóm H

| Thuộc tính                 | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu     |
| -------------------------- | --------------- | -------- | ------------------- | --------------------- |
| `id`                       | `uuid`          | Có       | `gen_random_uuid()` | PK                    |
| `event_update_id`          | `uuid`          | Có       | —                   | FK → event_updates.id |
| `urgency_score`            | `numeric`       | Có       | —                   | —                     |
| `impact_score`             | `numeric`       | Có       | —                   | —                     |
| `novelty_score`            | `numeric`       | Có       | —                   | —                     |
| `verification_status`      | `text`          | Có       | —                   | —                     |
| `independent_source_count` | `integer`       | Có       | —                   | —                     |
| `is_critical`              | `boolean`       | Có       | —                   | —                     |
| `rationale`                | `text`          | Có       | —                   | —                     |
| `evidence_snapshot`        | `jsonb`         | Có       | `{}`                | —                     |
| `model_version`            | `text`          | Có       | —                   | —                     |
| `policy_version`           | `text`          | Có       | —                   | —                     |
| `assessed_at`              | `timestamptz`   | Có       | `now()`             | —                     |

### `user_alert_preferences` — nhóm M

| Thuộc tính                      | Kiểu PostgreSQL          | Bắt buộc | Mặc định | Khóa / tham chiếu |
| ------------------------------- | ------------------------ | -------- | -------- | ----------------- |
| `user_id`                       | `uuid`                   | Có       | —        | PK; FK → users.id |
| `enabled`                       | `boolean`                | Có       | `false`  | —                 |
| `max_alerts_per_day`            | `integer`                | Có       | —        | —                 |
| `min_interval_minutes`          | `integer`                | Có       | —        | —                 |
| `quiet_hours_start`             | `time without time zone` | Không    | —        | —                 |
| `quiet_hours_end`               | `time without time zone` | Không    | —        | —                 |
| `critical_bypasses_quiet_hours` | `boolean`                | Có       | `false`  | —                 |

### `user_topic_alert_settings` — nhóm M

| Thuộc tính | Kiểu PostgreSQL | Bắt buộc | Mặc định | Khóa / tham chiếu  |
| ---------- | --------------- | -------- | -------- | ------------------ |
| `user_id`  | `uuid`          | Có       | —        | PK; FK → users.id  |
| `topic_id` | `uuid`          | Có       | —        | PK; FK → topics.id |
| `enabled`  | `boolean`       | Có       | `false`  | —                  |

### `breaking_alerts` — nhóm M

| Thuộc tính                | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu         |
| ------------------------- | --------------- | -------- | ------------------- | ------------------------- |
| `id`                      | `uuid`          | Có       | `gen_random_uuid()` | PK                        |
| `user_id`                 | `uuid`          | Có       | —                   | FK → users.id             |
| `event_update_id`         | `uuid`          | Có       | —                   | FK → event_updates.id     |
| `assessment_id`           | `uuid`          | Có       | —                   | FK → event_assessments.id |
| `status`                  | `text`          | Có       | `pending`           | —                         |
| `matched_topics_snapshot` | `jsonb`         | Có       | `[]`                | —                         |
| `content_snapshot`        | `jsonb`         | Không    | —                   | —                         |
| `policy_snapshot`         | `jsonb`         | Có       | `{}`                | —                         |
| `eligible_at`             | `timestamptz`   | Có       | —                   | —                         |
| `expires_at`              | `timestamptz`   | Không    | —                   | —                         |
| `cancellation_reason`     | `text`          | Không    | —                   | —                         |

### `newsletters` — nhóm M

| Thuộc tính               | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu          |
| ------------------------ | --------------- | -------- | ------------------- | -------------------------- |
| `id`                     | `uuid`          | Có       | `gen_random_uuid()` | PK                         |
| `user_id`                | `uuid`          | Có       | —                   | FK → users.id              |
| `newsletter_config_id`   | `uuid`          | Có       | —                   | FK → newsletter_configs.id |
| `scheduled_for`          | `timestamptz`   | Có       | —                   | —                          |
| `status`                 | `text`          | Có       | `pending`           | —                          |
| `config_revision`        | `integer`       | Có       | —                   | —                          |
| `config_snapshot`        | `jsonb`         | Có       | `{}`                | —                          |
| `subject`                | `text`          | Không    | —                   | —                          |
| `overview`               | `text`          | Không    | —                   | —                          |
| `estimated_read_minutes` | `numeric`       | Không    | —                   | —                          |
| `content_cutoff_at`      | `timestamptz`   | Có       | —                   | —                          |
| `generated_at`           | `timestamptz`   | Không    | —                   | —                          |
| `skip_reason`            | `text`          | Không    | —                   | —                          |
| `last_error`             | `text`          | Không    | —                   | —                          |

### `newsletter_items` — nhóm H

| Thuộc tính                   | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu       |
| ---------------------------- | --------------- | -------- | ------------------- | ----------------------- |
| `id`                         | `uuid`          | Có       | `gen_random_uuid()` | PK                      |
| `newsletter_id`              | `uuid`          | Có       | —                   | FK → newsletters.id     |
| `event_update_id`            | `uuid`          | Có       | —                   | FK → event_updates.id   |
| `event_summary_id`           | `uuid`          | Có       | —                   | FK → event_summaries.id |
| `primary_topic_id`           | `uuid`          | Có       | —                   | FK → topics.id          |
| `position`                   | `integer`       | Có       | —                   | —                       |
| `content_snapshot`           | `jsonb`         | Có       | —                   | —                       |
| `source_references_snapshot` | `jsonb`         | Có       | —                   | —                       |

### `user_event_deliveries` — nhóm M

| Thuộc tính               | Kiểu PostgreSQL | Bắt buộc | Mặc định   | Khóa / tham chiếu         |
| ------------------------ | --------------- | -------- | ---------- | ------------------------- |
| `user_id`                | `uuid`          | Có       | —          | PK; FK → users.id         |
| `event_update_id`        | `uuid`          | Có       | —          | PK; FK → event_updates.id |
| `email_delivery_id`      | `uuid`          | Có       | —          | FK → email_deliveries.id  |
| `status`                 | `text`          | Có       | `reserved` | —                         |
| `reserved_at`            | `timestamptz`   | Có       | `now()`    | —                         |
| `reservation_expires_at` | `timestamptz`   | Có       | —          | —                         |
| `submitted_at`           | `timestamptz`   | Không    | —          | —                         |

### `email_templates` — nhóm M

| Thuộc tính         | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| ------------------ | --------------- | -------- | ------------------- | ----------------- |
| `id`               | `uuid`          | Có       | `gen_random_uuid()` | PK                |
| `template_key`     | `text`          | Có       | —                   | —                 |
| `language`         | `text`          | Có       | —                   | —                 |
| `subject_template` | `text`          | Có       | —                   | —                 |
| `html_template`    | `text`          | Có       | —                   | —                 |
| `text_template`    | `text`          | Có       | —                   | —                 |
| `status`           | `text`          | Có       | `inactive`          | —                 |

### `email_deliveries` — nhóm M

| Thuộc tính                 | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu          |
| -------------------------- | --------------- | -------- | ------------------- | -------------------------- |
| `id`                       | `uuid`          | Có       | `gen_random_uuid()` | PK                         |
| `user_id`                  | `uuid`          | Có       | —                   | FK → users.id              |
| `newsletter_id`            | `uuid`          | Không    | —                   | FK → newsletters.id        |
| `breaking_alert_id`        | `uuid`          | Không    | —                   | FK → breaking_alerts.id    |
| `access_token_id`          | `uuid`          | Không    | —                   | FK → user_access_tokens.id |
| `email_template_id`        | `uuid`          | Có       | —                   | FK → email_templates.id    |
| `recipient_email_snapshot` | `text`          | Có       | —                   | —                          |
| `subject_snapshot`         | `text`          | Có       | —                   | —                          |
| `rendered_html`            | `text`          | Có       | —                   | —                          |
| `rendered_text`            | `text`          | Có       | —                   | —                          |
| `idempotency_key`          | `text`          | Có       | —                   | UNIQUE                     |
| `status`                   | `text`          | Có       | `queued`            | —                          |
| `provider`                 | `text`          | Có       | —                   | —                          |
| `provider_message_id`      | `text`          | Không    | —                   | —                          |
| `next_attempt_at`          | `timestamptz`   | Không    | —                   | —                          |
| `submitted_at`             | `timestamptz`   | Không    | —                   | —                          |
| `delivered_at`             | `timestamptz`   | Không    | —                   | —                          |
| `last_error`               | `text`          | Không    | —                   | —                          |

### `email_delivery_attempts` — nhóm M

| Thuộc tính            | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu        |
| --------------------- | --------------- | -------- | ------------------- | ------------------------ |
| `id`                  | `uuid`          | Có       | `gen_random_uuid()` | PK                       |
| `email_delivery_id`   | `uuid`          | Có       | —                   | FK → email_deliveries.id |
| `attempt_number`      | `integer`       | Có       | —                   | —                        |
| `started_at`          | `timestamptz`   | Có       | `now()`             | —                        |
| `finished_at`         | `timestamptz`   | Không    | —                   | —                        |
| `status`              | `text`          | Có       | `started`           | —                        |
| `provider_message_id` | `text`          | Không    | —                   | —                        |
| `error_code`          | `text`          | Không    | —                   | —                        |
| `error_message`       | `text`          | Không    | —                   | —                        |

### `email_provider_events` — nhóm M

| Thuộc tính            | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu        |
| --------------------- | --------------- | -------- | ------------------- | ------------------------ |
| `id`                  | `uuid`          | Có       | `gen_random_uuid()` | PK                       |
| `provider`            | `text`          | Có       | —                   | —                        |
| `provider_event_id`   | `text`          | Có       | —                   | —                        |
| `email_delivery_id`   | `uuid`          | Không    | —                   | FK → email_deliveries.id |
| `provider_message_id` | `text`          | Không    | —                   | —                        |
| `event_type`          | `text`          | Có       | —                   | —                        |
| `occurred_at`         | `timestamptz`   | Có       | —                   | —                        |
| `received_at`         | `timestamptz`   | Có       | `now()`             | —                        |
| `payload`             | `jsonb`         | Có       | —                   | —                        |
| `processed_at`        | `timestamptz`   | Không    | —                   | —                        |

### `background_jobs` — nhóm M

| Thuộc tính        | Kiểu PostgreSQL | Bắt buộc | Mặc định            | Khóa / tham chiếu |
| ----------------- | --------------- | -------- | ------------------- | ----------------- |
| `id`              | `uuid`          | Có       | `gen_random_uuid()` | PK                |
| `job_type`        | `text`          | Có       | —                   | —                 |
| `payload`         | `jsonb`         | Có       | `{}`                | —                 |
| `idempotency_key` | `text`          | Có       | —                   | UNIQUE            |
| `status`          | `text`          | Có       | `pending`           | —                 |
| `run_after`       | `timestamptz`   | Có       | `now()`             | —                 |
| `attempt_count`   | `integer`       | Có       | `0`                 | —                 |
| `max_attempts`    | `integer`       | Có       | —                   | —                 |
| `locked_by`       | `text`          | Không    | —                   | —                 |
| `locked_until`    | `timestamptz`   | Không    | —                   | —                 |
| `last_error`      | `text`          | Không    | —                   | —                 |
| `correlation_id`  | `uuid`          | Không    | —                   | —                 |

### Ràng buộc bổ sung và tính nhất quán

- Số thứ tự `version`, `summary_version`, `revision`, `config_revision`, `attempt_number` và `position` phải > 0. `display_order`, `attempt_count`, `consecutive_failures`, `independent_source_count` phải ≥ 0; `max_attempts` > 0.
- `max_total_stories`, `max_stories_per_newsletter`, `target_read_minutes` (nếu có) > 0. `max_alerts_per_day` > 0, `min_interval_minutes` ≥ 0; chưa có số mặc định cho hai trường này. Khi tạo tùy chọn cảnh báo phải truyền chính sách đã chốt, không diễn giải NULL là vô hạn.
- Điểm `relevance_score`, `urgency_score`, `impact_score`, `novelty_score` nằm trong [0, 1]; `estimated_read_minutes` ≥ 0 nếu có.
- `quiet_hours_start` và `quiet_hours_end` cùng null hoặc cùng có giá trị và khác nhau. `expires_at` của token > `created_at`; lease/reservation hết hạn sau lúc bắt đầu. `finished_at` không trước `started_at`.
- Múi giờ phải là IANA zone hợp lệ. Ngôn ngữ dùng mã BCP 47 đã được hệ thống hỗ trợ; chưa xác định được ngôn ngữ bài/nguồn thì dùng NULL. Các trường ngôn ngữ bắt buộc phải truyền mã thực tế.
- `articles.extraction_method` nhận `rss`, `api`, `web`. `content_status='extracted'` yêu cầu `content_text` không rỗng, `content_hash`, `extracted_at`; các trạng thái còn lại không giữ `content_text`. `unavailable` phải có `extraction_error`; `expired` chỉ áp dụng sau khi đã xóa nội dung. Lỗi trích xuất không đồng nghĩa lỗi AI trong `articles.status`.
- `content_expires_at` nullable khi chưa có nội dung hoặc chưa ấn định thời hạn; worker phải áp dụng chính sách retention khi lưu văn bản, không xem NULL là quyền giữ vô hạn. Hash nội dung không bị xóa theo văn bản. `fetched_at` chỉ ghi sau khi nhận được phản hồi nội dung; metadata RSS có thể chưa có thời điểm lấy trang riêng.
- `publisher_group_key` nullable nghĩa chưa biết nhóm nhà xuất bản; không tự tính mỗi nguồn chưa rõ là một nguồn độc lập đã xác minh.
- `event_updates` ở `ready` cần facts và nguồn đủ điều kiện; `event_summaries`/`event_assessments` là kết quả hoàn chỉnh, không tạo bản rỗng để làm trạng thái chờ. Job chịu trách nhiệm lưu lỗi xử lý.
- `breaking_alerts.content_snapshot` có thể null khi `pending`, bắt buộc có khi `ready`. Snapshot topic là mảng; facts/config/policy/content snapshot là object; nguồn trích dẫn là mảng có `article_id`, `source_name`, `title`, `original_url`, `canonical_url`. Snapshot giữ văn bản/nguồn thực tế, không chỉ giữ ID.
- `newsletters` ở `ready` phải có `subject`, `generated_at` và ít nhất một item; `skipped` phải có `skip_reason`. Topic chính của item phải nằm trong cấu hình snapshot và liên quan đến sự kiện.
- Tạo `email_deliveries` trước hoặc cùng transaction với `user_event_deliveries` vì reservation tham chiếu delivery bắt buộc. Email ở `queued` có thể chưa được worker gửi; chỉ enqueue job sau khi đủ item/reservation và snapshot. Khi nhường tin cho cảnh báo, cập nhật cả reservation và nội dung email chưa phát hành trong cùng transaction.
- Các cột `newsletter_id`, `breaking_alert_id`, `access_token_id` trong `email_deliveries` có unique riêng và CHECK đúng một tham chiếu khác NULL. Kiểm tra mục đích token khớp template; liên kết token chỉ phục vụ email xác minh/quản lý, không tự gửi email cho token hủy đăng ký.
- Retry dùng nguyên email đã render, không render lại theo template vừa sửa. Không lưu HTML chứa token truy cập dưới dạng công khai; hạn chế truy cập snapshot và dọn theo retention.
- Job `running` phải có `locked_by`, `locked_until`. `correlation_id` là UUID truy vết, không phải FK. Attempt `started` cho phép `finished_at` null; các kết quả hoàn tất phải có `finished_at`.
- `user_event_deliveries` là trạng thái chống trùng hiện tại, có thể chuyển `released` về `reserved` với delivery mới trong transaction. Lịch sử các lần gửi vẫn nằm ở delivery/attempts; không xóa bản đã `submitted` chỉ vì hết lease.

Việc chuẩn hóa thuộc tính không thay đổi các quyết định sản phẩm còn mở ở phần 17. Chưa tạo bảng hoặc migration trong database thực tế.
