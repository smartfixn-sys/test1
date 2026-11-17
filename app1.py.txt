import os
import sqlite3
import uuid
from datetime import date
from PIL import Image
import streamlit as st

# إعداد الصفحة
st.set_page_config(page_title="أرشيف المؤمن عليهم", page_icon="📁", layout="wide")

# إنشاء مجلد لحفظ الصور المرفوعة
UPLOAD_DIR = "uploaded_images"
os.makedirs(UPLOAD_DIR, exist_ok=True)

# إعداد قاعدة البيانات بالحقول الجديدة
def init_db():
    with sqlite3.connect("archive.db") as conn:
        cursor = conn.cursor()
        cursor.execute("""
            CREATE TABLE IF NOT EXISTS archive (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                national_id TEXT NOT NULL,
                phone TEXT NOT NULL,
                insurance_type TEXT NOT NULL,
                start_date TEXT NOT NULL,
                end_date TEXT NOT NULL,
                image_path TEXT NOT NULL
            )
        """)
        conn.commit()

init_db()

st.title("📁 برنامج الأرشيف الإلكتروني للأسماء والتأمين")
st.markdown("إدارة سجلات العملاء، بيانات التأمين، والمرفقات بسهولة.")

# --- الشريط الجانبي للإضافة ---
st.sidebar.header("➕ إضافة سجل جديد")

name_input = st.sidebar.text_input("الاسم الكامل:")
national_id_input = st.sidebar.text_input("رقم الهوية / الإقامة:")
phone_input = st.sidebar.text_input("رقم الهاتف:")

insurance_options = ["تأمين شامل", "تأمين ضد الغير", "تأمين طبي", "تأمين سفر", "أخرى"]
insurance_type_input = st.sidebar.selectbox("نوع التأمين:", insurance_options)

col_date1, col_date2 = st.sidebar.columns(2)
with col_date1:
    start_date_input = st.date_input("تاريخ البداية:", value=date.today())
with col_date2:
    end_date_input = st.date_input("تاريخ النهاية:", value=date.today())

uploaded_file = st.sidebar.file_uploader("اختر صورة الوثيقة / الشخصية:", type=["jpg", "png", "jpeg"])

if st.sidebar.button("حفظ في الأرشيف", type="primary"):
    if name_input and national_id_input and phone_input and uploaded_file:
        if end_date_input < start_date_input:
            st.sidebar.error("خطأ: تاريخ النهاية يجب أن يكون بعد تاريخ البداية!")
        else:
            # توليد اسم فريد للصورة
            file_extension = os.path.splitext(uploaded_file.name)[1]
            unique_filename = f"{uuid.uuid4()}{file_extension}"
            image_path = os.path.join(UPLOAD_DIR, unique_filename)
            
            # حفظ الصورة
            with open(image_path, "wb") as f:
                f.write(uploaded_file.getbuffer())
            
            # حفظ البيانات في قاعدة البيانات
            with sqlite3.connect("archive.db") as conn:
                cursor = conn.cursor()
                cursor.execute("""
                    INSERT INTO archive (name, national_id, phone, insurance_type, start_date, end_date, image_path) 
                    VALUES (?, ?, ?, ?, ?, ?, ?)
                """, (
                    name_input, 
                    national_id_input, 
                    phone_input, 
                    insurance_type_input, 
                    str(start_date_input), 
                    str(end_date_input), 
                    image_path
                ))
                conn.commit()
            
            st.sidebar.success(f"تم حفظ سجل '{name_input}' بنجاح! ✅")
            st.rerun()
    else:
        st.sidebar.error("الرجاء تعبئة جميع الحقول واختيار صورة.")

# --- قسم البحث والعرض ---
st.header("📋 الأرشيف الحالي")
search_query = st.text_input("🔍 ابحث بالاسم أو رقم الهوية:", placeholder="اكتب الاسم أو رقم الهوية للبحث...")

# استرجاع البيانات بناءً على البحث
with sqlite3.connect("archive.db") as conn:
    cursor = conn.cursor()
    if search_query:
        cursor.execute("""
            SELECT id, name, national_id, phone, insurance_type, start_date, end_date, image_path 
            FROM archive 
            WHERE name LIKE ? OR national_id LIKE ?
        """, ('%' + search_query + '%', '%' + search_query + '%'))
    else:
        cursor.execute("""
            SELECT id, name, national_id, phone, insurance_type, start_date, end_date, image_path 
            FROM archive
        """)
    records = cursor.fetchall()

if not records:
    st.info("لا توجد نتائج مطابقة في الأرشيف.")
else:
    st.write(f"عدد السجلات المعروضة: **{len(records)}**")
    st.markdown("---")
    
    for record_id, name, nat_id, phone, ins_type, s_date, e_date, img_path in records:
        col1, col2, col3, col4 = st.columns([1, 2.5, 2.5, 1])
        
        with col1:
            if os.path.exists(img_path):
                try:
                    img = Image.open(img_path)
                    st.image(img, width=110)
                except Exception:
                    st.warning("صورة تالفة")
            else:
                st.warning("لا توجد صورة")
                
        with col2:
            st.subheader(f"👤 {name}")
            st.write(f"**🆔 رقم الهوية:** {nat_id}")
            st.write(f"**📞 الهاتف:** {phone}")
            
        with col3:
            st.write("") # مسافة للتنسيق
            st.write(f"**🛡️ نوع التأمين:** {ins_type}")
            st.write(f"**📅 من:** {s_date}  |  **إلى:** {e_date}")
            
            # تنبيه تلقائي في حال انتهاء صلاحية التأمين
            if date.fromisoformat(e_date) < date.today():
                st.error("⚠️ التأمين منتهي الصلاحية")
            else:
                st.success("✅ التأمين ساري المفعول")
            
        with col4:
            st.write("")
            if st.button("🗑️ حذف", key=f"del_{record_id}"):
                if os.path.exists(img_path):
                    try:
                        os.remove(img_path)
                    except Exception:
                        pass
                
                with sqlite3.connect("archive.db") as conn:
                    cursor = conn.cursor()
                    cursor.execute("DELETE FROM archive WHERE id = ?", (record_id,))
                    conn.commit()
                
                st.success("تم الحذف!")
                st.rerun()
                
        st.markdown("---")

