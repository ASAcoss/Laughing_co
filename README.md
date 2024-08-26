import streamlit as st
from pymongo import MongoClient
from datetime import datetime, timedelta
import bcrypt


# MongoDB 클라이언트 설정
client = MongoClient('mongodb+srv://asaasacoss:<db_password>@cluster0.eckjj.mongodb.net/?retryWrites=true&w=majority&appName=Cluster0')
db = client['오아시스데이터']

# 데이터베이스의 각 컬렉션 선택
requests_collection = db['Requests']
users_collection = db['Users']
transactions_collection = db['Transactions']
coupons_collection = db['Coupons']
inquiries_collection = db['Inquiries']

def login_user(username, password):
    # MongoDB에서 사용자 검색
    user = users_collection.find_one({"username": username})
    if user and bcrypt.checkpw(password.encode('utf-8'), user['password']):
        return True, user
    return False, None

def register_user(username, password):
    # 중복 사용자 이름 검사
    if users_collection.find_one({"username": username}):
        return False, "사용자 이름이 이미 존재합니다."
    # 비밀번호 해싱
    hashed_password = bcrypt.hashpw(password.encode('utf-8'), bcrypt.gensalt())
    # 새 사용자 저장
    users_collection.insert_one({"username": username, "password": hashed_password})
    return True, "회원가입 성공"

# 초기 세션 상태 설정
if 'logged_in' not in st.session_state:
    st.session_state['logged_in'] = False


if 'active_tab' not in st.session_state:
    st.session_state['active_tab'] = "수거 품목"

# 로그인 및 회원가입 버튼
def authenticate_user():
    col1, col2 = st.columns([1, 1])
    with col1:
        username = st.text_input("사용자 이름", key="login_username")
        password = st.text_input("비밀번호", type="password", key="login_password")
        if st.button("로그인"):
            logged_in, user = login_user(username, password)
            if logged_in:
                st.session_state['logged_in'] = True
                st.session_state['user_id'] = user['_id']  # MongoDB 문서의 ID를 저장
                st.success("로그인 성공!")
            else:
                st.error("로그인 실패. 사용자 이름 또는 비밀번호를 확인해주세요.")

    with col2:
        new_username = st.text_input("새 사용자 이름", key="register_username")
        new_password = st.text_input("새 비밀번호", type="password", key="register_password")
        if st.button("회원가입"):
            success, message = register_user(new_username, new_password)
            if success:
                st.success(message)
            else:
                st.error(message)
def switch_to_next_tab():
    if st.session_state['active_tab'] == "수거 품목":
        st.session_state['active_tab'] = "수거 요청"
    elif st.session_state['active_tab'] == "수거 요청":
        st.session_state['active_tab'] = "마이 페이지"
    else:
        st.session_state['active_tab'] = "수거 품목"
authenticate_user()

# 로그인 검증 후 메인 앱 코드
if 'logged_in' in st.session_state:
    if 'user_id' in st.session_state:
        user_id = st.session_state['user_id']
        # MongoDB에서 해당 사용자의 수거 요청 내역 조회
        pickup_requests = list(requests_collection.find({"userId": user_id}))
        if pickup_requests:
            st.subheader("수거 요청 내역")
            for request in pickup_requests:
                st.write(f"ID: {request['requestId']}, 항목: {request['items']}, 상태: {request['status']}, 주소: {request['address']}")
        else:
            st.write("수거 요청 내역이 없습니다.")
    else:
        st.error("사용자 인증 정보를 찾을 수 없습니다.")
else:
    st.info("로그인을 해주세요.")
    
if st.session_state['logged_in']:
    # 탭 정의
    tabs = ["수거 품목", "수거 요청", "마이 페이지"]
    # 탭 선택에 따라 UI 렌더링
    if st.session_state['active_tab'] == "수거 품목":
        tab1, tab2, tab3 = st.tabs(tabs)
        with tab1:
            st.header("수거 품목")

            price = {
            "종이류": 1000.0,  # 1kg당 1000.0원
            "병류": 1000.0,     # 1kg당 1000.0원
            "고철류": 1000.0,   # 1kg당 1000.0원
            "캔류": 1000.0,     # 1kg당 1000.0원
            "비닐류": 1000.0,   # 1kg당 1000.0원
            "스티로폼류": 1000.0, # 1kg당 1000.0원
            "플라스틱류": 1000.0, # 1kg당 1000.0원
            "의류": 1000.0    # 1kg당 1000.0원
            }
            st.markdown("""
           <style>
           .divider {
           height: 100%;
           width: 3px;
           background-color: green;
           margin: 0px 10px;
           }
           </style>
           """, unsafe_allow_html=True)
            def add_to_cart(item_name, weight_key, price_per_kg, button_key):
                weight = st.number_input(f"{item_name} 무게 입력 (kg)", min_value=0.0, step=0.1, key=weight_key)
                if st.button(f"추가하기", key=button_key):
                    if weight > 0:
                        weight = round(weight,2)
                        price = weight * price_per_kg
                    st.session_state.cart = [item for item in st.session_state.cart if not item.startswith(item_name)]
                    st.session_state.cart.append(f"{item_name}: {weight} kg          + {price} 원")
                    if item_name in st.session_state.price_tracker:
                        st.session_state.total_price -= st.session_state.price_tracker[item_name]  # 기존 가격 제거
                    st.session_state.total_price += price
                    st.session_state.price_tracker[item_name] = price
            # 첫 번째 열
            col1, col2 = st.columns(2)
            with col1:
                with st.expander("종이류"):
                    st.write("종이 1kg 당 1,000원")
                    add_to_cart("종이류", "paper_weight", price["종이류"], "add_paper")
        
            with col2:
                with st.expander("플라스틱류"):
                    st.write("플라스틱 1kg 당 1,000원")
                    add_to_cart("플라스틱류", "plastic_weight", price["플라스틱류"], "add_plastic")
    
    # 두 번째 열
            col3, col4 = st.columns(2)
            with col3:
                with st.expander("비닐류"):
                    st.write("비닐 1kg 당 1,000원")
                    add_to_cart("비닐류", "vinyl_weight", price["비닐류"], "add_vinyl")
        
            with col4:
                with st.expander("캔류"):
                    st.write("캔 1kg 당 1,000원")
                    add_to_cart("캔류", "can_weight", price["캔류"], "add_can")
    
    # 세 번째 열
            col5, col6 = st.columns(2)
            with col5:
                with st.expander("병류"):
                    st.write("병 1kg 당 1,000원")
                    add_to_cart("병류", "bottle_weight", price["병류"], "add_bottle")
    
            with col6:
                with st.expander("스티로폼류"):
                    st.write("스티로폼 1kg 당 1,000원")
                    add_to_cart("스티로폼류", "styrofoam_weight", price["스티로폼류"], "add_styrofoam")
    
    # 네 번째 열
            col7, col8 = st.columns(2)
            with col7:
                with st.expander("고철류"):
                    st.write("고철 1kg 당 1,000원")
                    add_to_cart("고철류", "metal_weight", price["고철류"], "add_metal")
        
            with col8:
                with st.expander("의류"):
                    st.write("의류 1kg 당 1,000원")
                    add_to_cart("의류", "clothes_weight", price["의류"], "add_clothes")
            
    # 장바구니 표시
            st.divider()
            with st.expander("**나의 장바구니🛒**", expanded=True):
                st.write("**장바구니에 담긴 품목**")
                if st.session_state.cart:
                    for item in st.session_state.cart:
                        st.write(f"- {item}")
                    st.write(f"**총 환급액: {st.session_state.total_price:.2f} 원**")
                else:
                    st.write("장바구니가 비어 있습니다.")
            st.markdown("""
        <style>
        .custom-button {
           display: inline-block;
           padding: 10px 20px;
           font-size: 20px;
           color: white;
           background-color: green;
           border: 2px solid green;
           border-radius: 5px;
           text-align: center;
           cursor: pointer;
           }
        .custom-button:hover {
        background-color: darkgreen;
        border-color: darkgreen;
        }
        </style>
        """, unsafe_allow_html=True)
    # 버튼 생성
            if st.button('다음 단계'):
            #st.write(".")
                switch_to_next_tab()
#if st.markdown('<div class="custom-button">다음 단계</div>', unsafe_allow_html=True):

    elif st.session_state['active_tab'] == "수거 요청":
            tab1, tab2, tab3 = st.tabs(tabs)
            with tab2:
                st.header("수거 요청")
                st.write("수거할 쓰레기 종류를 선택하세요 (복수 선택 가능):")
                waste_types = st.multiselect(
                    "종류 선택", 
                    options=["종이류", "병류", "고철류", "캔류", "비닐류", "스티로폼류", "플라스틱류", "의류" ]
                )

                address = st.text_input("쓰레기 수거 주소:")

                today = datetime.now().date()
                min_pickup_date = today + timedelta(days=2)

                pickup_date = st.date_input(
                    "수거 희망 날짜를 선택하세요:",
                    min_value = min_pickup_date,
                    value = min_pickup_date
                )

                st.divider()
                st.write("### 사용자 요청 정보")
                if waste_types :
                    st.write(f"- 수거할 쓰레기 종류: {', '.join(waste_types)}")
                else: 
                    st.write("-수거할 쓰레기 종류를 선택하지 않았습니다.")
    
                if address: 
                    st.write(f"- 수거 주소: {address}")
                else:
                    st.write("-수거 주소를 입력하지 않았습니다.")
    
                st.write(f"-수거 희망 날짜: {pickup_date}")
                if st.markdown('<div class="custom-button">수거 요청</div>', unsafe_allow_html=True):
                    st.write(".")
        elif st.session_state['active_tab'] == "마이 페이지":
            tab1, tab2, tab3 = st.tabs(tabs)
            with tab3:
                if 'page' in st.session_state:
                    current_page = st.session_state['page']
                else:
                    st.session_state['page'] = 'default_page'
                    current_page = 'default_page'
                st.header("마이 페이지")

                if 'logged_in' in st.session_state and 'user_id' in st.session_state:
                    user_id = st.session_state['user_id']

        # 사용자 정보 조회
                    user_info = users_collection.find_one({"userId": user_id})

        # 메뉴 아이템 생성
                    menu_items = ["사용자 정보", "수거 요청 내역", "주소 관리", "환급 내역", "쿠폰함", "1:1 문의"]

                    for item in menu_items:
                        with st.expander(label=f"**{item}**", expanded=False):
                            if item == "사용자 정보":
                                if user_info:
                                    st.write(f"이름: {user_info['name']}")
                                    st.write(f"가입일: {user_info['join_date']}")  # 예시: 추가적인 사용자 정보 표시
                                else:
                                    st.error("사용자 정보를 찾을 수 없습니다.")
                            elif item == "수거 요청 내역":
                                pickup_requests = list(requests_collection.find({"userId": user_id}))
                                if pickup_requests:
                                    st.subheader("수거 요청 내역")
                                for request in pickup_requests:
                                    st.write(f"ID: {request['requestId']}, 항목: {request['items']}, 상태: {request['status']}, 주소: {request['address']}")
                                else:
                                    st.write("수거 요청 내역이 없습니다.")
                            elif item == "주소 관리":
                                user_info = users_collection.find_one({"userId": user_id})
                                if user_info:
                                    current_address = user_info['address']
                                    new_address = st.text_input("새 주소 입력", value=current_address)
                                    if st.button("주소 업데이트"):
                                        if new_address:
                                            users_collection.update_one(
                                                {"userId": user_id},
                                                {"$set": {"address": new_address}}
                                             )
                                            st.success("주소가 업데이트되었습니다.")
                                        else:
                                            st.warning("새로운 주소를 입력해주세요.")
                                    else:
                                        st.write("사용자 정보가 없습니다.")

                                elif item == "1:1 문의":
                                    inquiry = st.text_area("문의 내용을 입력하세요")
                                    if st.button("문의하기"):
                                        if inquiry:
                                            inquiries_collection.insert_one({
                                                "userId": user_id,
                                                "inquiry": inquiry,
                                                "createdAt": datetime.now()
                                            })
                                            st.success("문의가 등록되었습니다.")
                                    else:
                                        st.warning("문의 내용을 입력해주세요.")
                                elif item == "환급 내역":
                                    refund_history = list(transactions_collection.find({"userId": user_id}))
                                    if refund_history:
                                        for refund in refund_history:
                                            st.write(f"거래 ID: {refund['transactionId']}, 날짜: {refund['date']}, 상태: {refund['status']}")
                                    else:
                                        st.write("환급 내역이 없습니다.")
                                elif item == "쿠폰함":
                                    coupons = list(coupons_collection.find({"userId": user_id}))
                                    if coupons:
                                        for coupon in coupons:
                                            st.write(f"쿠폰: {coupon['description']}, 유효기간: {coupon['expiry_date']}")
                                else:
                                    st.write("쿠폰이 없습니다.")


