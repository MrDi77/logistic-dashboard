# logistic_dashboard.py
import streamlit as st
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
from datetime import datetime, timedelta
import googlemaps
import json
import os

# --- Настройка страницы ---
st.set_page_config(page_title="🚚 SmartLogistic", layout="wide")
st.title("🚚 SmartLogistic: Планирование подачи грузовиков")

# --- Инициализация session_state ---
default_types = ["еврофура", "малотоннажник", "рефрижератор", "тент", "изотерм"]

if 'vehicle_types_list' not in st.session_state:
    st.session_state.vehicle_types_list = default_types.copy()

if 'vehicles' not in st.session_state:
    st.session_state.vehicles = pd.DataFrame(columns=["id", "type", "max_trips", "depot_return_time_min"])

if 'stores' not in st.session_state:
    st.session_state.stores = pd.DataFrame(columns=["id", "name", "unloading_time", "doc_time", "open", "close", "allowed_vehicle_types"])

if 'distances' not in st.session_state:
    st.session_state.distances = pd.DataFrame(columns=["to", "duration_min"])

if 'assignments' not in st.session_state:
    st.session_state.assignments = []

if 'result_df' not in st.session_state:
    st.session_state.result_df = pd.DataFrame()

if 'gmaps_api_key' not in st.session_state:
    st.session_state.gmaps_api_key = ""

if 'depot_address' not in st.session_state:
    st.session_state.depot_address = "Москва, Волгоградский проспект, 1"

if 'store_addresses' not in st.session_state:
    st.session_state.store_addresses = {}

if 'need_recalc' not in st.session_state:
    st.session_state.need_recalc = False

# --- Вспомогательные функции ---
def time_to_minutes(t):
    return t.hour * 60 + t.minute

def minutes_to_time_str(m):
    h = (m // 60) % 24
    mi = m % 60
    return f"{int(h):02d}:{int(mi):02d}"

def run_optimization():
    assignments = st.session_state.assignments
    if not assignments:
        return

    results = []
    current_time = 1320  # 22:00 в минутах

    for a in assignments:
        vid = a["vehicle_id"]
        vtype = a["vehicle_type"]
        sid = a["store_id"]
        trip_num = a["trip_order"]

        vehicle = st.session_state.vehicles[st.session_state.vehicles["id"] == vid].iloc[0]
        store = st.session_state.stores[st.session_state.stores["id"] == sid].iloc[0]
        distance_row = st.session_state.distances[st.session_state.distances["to"] == sid]
        travel_time = distance_row["duration_min"].iloc[0] if len(distance_row) > 0 else 45
        load_time = 30
        unload_time = store["unloading_time"]
        doc_time = store["doc_time"]
        win_start = store["open"]
        win_end = store["close"]

        start_load = current_time
        end_load = start_load + load_time
        arrival = end_load + travel_time
        waiting = max(0, arrival - win_start)
        departure = arrival + unload_time + doc_time
        return_time = departure + travel_time

        results.append({
            "Машина": vid,
            "Тип": vtype,
            "Рейс": trip_num,
            "Магазин": sid,
            "Подача": minutes_to_time_str(start_load),
            "Погрузка до": minutes_to_time_str(end_load),
            "Прибытие": minutes_to_time_str(arrival),
            "Разгрузка до": minutes_to_time_str(departure),
            "Возврат": minutes_to_time_str(return_time),
            "Ожидание (мин)": waiting,
            "Время в рейсе (мин)": (departure - start_load)
        })

        current_time = return_time

    st.session_state.result_df = pd.DataFrame(results)

# --- Боковая панель ---
with st.sidebar:
    st.header("🔧 Ввод данных")
    tab = st.radio("Раздел", [
        "📅 Настройки",
        "🏭 Склад",
        "🏪 Магазины",
        "🚛 Машины",
        "gMaps: Адреса",
        "🎯 Назначение",
        "🔄 Расчёт"
    ])

    if st.button("🔁 Пересчитать всё"):
        st.session_state.need_recalc = True

# --- Основной контент ---
if st.session_state.need_recalc:
    run_optimization()
    st.session_state.need_recalc = False
    st.rerun()

# === 1. Настройки типов ===
if tab == "📅 Настройки":
    st.header("Управление типами машин")
    new_type = st.text_input("Добавить тип")
    if st.button("Добавить"):
        if new_type.strip() and new_type not in st.session_state.vehicle_types_list:
            st.session_state.vehicle_types_list.append(new_type.strip())
            st.success(f"Тип '{new_type}' добавлен")

    del_type = st.selectbox("Удалить тип", [""] + st.session_state.vehicle_types_list)
    if st.button("Удалить") and del_type and del_type not in default_types:
        st.session_state.vehicle_types_list.remove(del_type)
        st.success(f"Тип '{del_type}' удалён")

    st.write("### Текущие типы:")
    st.write(", ".join(st.session_state.vehicle_types_list))

# === 2. Склад ===
elif tab == "🏭 Склад":
    st.header("Адрес склада")
    addr = st.text_input("Адрес", value=st.session_state.depot_address)
    if st.button("Сохранить"):
        st.session_state.depot_address = addr
        st.success("Адрес склада сохранён")

# === 3. Магазины ===
elif tab == "🏪 Магазины":
    st.header("Добавить магазин")
    with st.form("add_store"):
        col1, col2 = st.columns(2)
        with col1:
            sid = st.number_input("ID", min_value=1, step=1)
            name = st.text_input("Название")
        with col2:
            u_time = st.number_input("Разгрузка (мин)", value=30)
            d_time = st.number_input("Документы (мин)", value=10)

        col3, col4 = st.columns(2)
        with col3:
            open_t = st.time_input("Открывается", datetime.strptime("06:00", "%H:%M").time())
        with col4:
            close_t = st.time_input("Закрывается", datetime.strptime("10:00", "%H:%M").time())

        allowed_types = st.multiselect(
            "Разрешённые типы",
            options=["all"] + st.session_state.vehicle_types_list,
            default=["all"]
        )
        if "all" in allowed_types:
            allowed_types = ["all"]

        if st.form_submit_button("Добавить"):
            if sid in st.session_state.stores["id"].values:
                st.error("ID занят")
            else:
                new_s = pd.DataFrame([{
                    "id": sid,
                    "name": name,
                    "unloading_time": u_time,
                    "doc_time": d_time,
                    "open": time_to_minutes(open_t),
                    "close": time_to_minutes(close_t),
                    "allowed_vehicle_types": json.dumps(allowed_types)
                }])
                st.session_state.stores = pd.concat([st.session_state.stores, new_s], ignore_index=True)
                st.success("Магазин добавлен")

    st.dataframe(st.session_state.stores.assign(
        allowed_str=lambda x: x["allowed_vehicle_types"].apply(lambda y: ", ".join(json.loads(y)))
    ).drop(columns=["allowed_vehicle_types"]), use_container_width=True)

# === 4. Машины ===
elif tab == "🚛 Машины":
    st.header("Добавить машину")
    with st.form("add_vehicle"):
        col1, col2, col3 = st.columns(3)
        with col1:
            vid = st.number_input("ID", min_value=1, step=1)
        with col2:
            vtype = st.selectbox("Тип", st.session_state.vehicle_types_list)
        with col3:
            max_trips = st.number_input("Макс. рейсов", min_value=1, max_value=3, value=2)

        if st.form_submit_button("Добавить"):
            if vid in st.session_state.vehicles["id"].values:
                st.error("ID занят")
            else:
                new_v = pd.DataFrame([{
                    "id": vid,
                    "type": vtype,
                    "max_trips": max_trips,
                    "depot_return_time_min": 30
                }])
                st.session_state.vehicles = pd.concat([st.session_state.vehicles, new_v], ignore_index=True)
                st.success("Машина добавлена")

    st.dataframe(st.session_state.vehicles, use_container_width=True)

# === 5. Адреса и Google Maps ===
elif tab == "gMaps: Адреса":
    st.header("gMaps: Адреса магазинов")

    if not st.session_state.gmaps_api_key:
        st.warning("Введите API-ключ ниже")
    st.text_input("gMaps API-ключ", value=st.session_state.gmaps_api_key, type="password", key="temp_key")
    if st.button("Сохранить ключ"):
        st.session_state.gmaps_api_key = st.session_state.temp_key
        st.success("Ключ сохранён")

    st.text_input("Склад", value=st.session_state.depot_address, key="depot_temp")
    if st.button("Обновить адрес склада"):
        st.session_state.depot_address = st.session_state.depot_temp

    for _, s in st.session_state.stores.iterrows():
        addr = st.session_state.store_addresses.get(s["id"], "")
        new_addr = st.text_input(f"Магазин {s['id']} — {s['name']}", value=addr, key=f"addr_{s['id']}")
        st.session_state.store_addresses[s["id"]] = new_addr

elif tab == "🎯 Назначение":
    st.header("Назначить рейсы")
    vehicles = st.session_state.vehicles
    stores = st.session_state.stores.copy()
    stores["allowed_vehicle_types"] = stores["allowed_vehicle_types"].apply(json.loads)

    assignments = []
    for _, v in vehicles.iterrows():
        st.subheader(f"Машина {v['id']} ({v['type']})")
        allowed_stores = stores[
            (stores["allowed_vehicle_types"].apply(lambda x: "all" in x or v["type"] in x))
        ]
        selected = st.multiselect(
            "Магазины",
            options=allowed_stores["id"].tolist(),
            format_func=lambda x: f"{x} — {allowed_stores[allowed_stores['id']==x]['name'].iloc[0]}",
            key=f"ms_{v['id']}"
        )
        for i, sid in enumerate(selected):
            assignments.append({
                "vehicle_id": v["id"],
                "vehicle_type": v["type"],
                "store_id": sid,
                "trip_order": i + 1
            })

    if st.button("Сохранить назначения"):
        st.session_state.assignments = assignments
        st.success("Назначения сохранены!")

elif tab == "🔄 Расчёт":
    st.header("gMaps: Расчёт времени в пути")
    if st.session_state.gmaps_api_key:
        gmaps = googlemaps.Client(key=st.session_state.gmaps_api_key)
        if st.button("🚀 Рассчитать маршруты"):
            results = []
            for _, s in st.session_state.stores.iterrows():
                addr = st.session_state.store_addresses.get(s["id"], "").strip()
                if not addr:
                    continue
                try:
                    matrix = gmaps.distance_matrix(
                        origins=[st.session_state.depot_address],
                        destinations=[addr],
                        mode="driving",
                        departure_time=datetime.now()
                    )
                    if matrix["status"] == "OK":
                        elem = matrix["rows"][0]["elements"][0]
                        if elem["status"] == "OK":
                            dur_min = round(elem["duration"]["value"] / 60)
                            if s["id"] in st.session_state.distances["to"].values:
                                st.session_state.distances.loc[st.session_state.distances["to"] == s["id"], "duration_min"] = dur_min
                            else:
                                new_row = pd.DataFrame([{"to": s["id"], "duration_min": dur_min}])
                                st.session_state.distances = pd.concat([st.session_state.distances, new_row], ignore_index=True)
                            results.append({"Магазин": s["id"], "Время": f"{dur_min} мин"})
                except Exception as e:
                    results.append({"Магазин": s["id"], "Время": "Ошибка"})
            st.dataframe(pd.DataFrame(results), use_container_width=True)
    else:
        st.warning("Введите API-ключ")

# --- Основной дашборд ---
st.divider()
st.header("📊 Дашборд")

col1, col2 = st.columns([2, 1])

# === Матрица подачи ===
with col1:
    st.subheader("Тепловая карта загрузки")
    if not st.session_state.result_df.empty:
        df = st.session_state.result_df
        hours = pd.date_range("2025-04-05 22:00", "2025-04-06 10:00", freq="H")
        hour_labels = [h.strftime("%H:%M") for h in hours]
        matrix = pd.DataFrame(index=sorted(df['Машина'].unique()), columns=hour_labels, data=0)

        for _, row in df.iterrows():
            vid = row['Машина']
            start = datetime.strptime(row['Подача'], "%H:%M")
            return_t = datetime.strptime(row['Возврат'], "%H:%M")
            if start.hour < 10:
                start += timedelta(days=1)
            if return_t.hour < 10:
                return_t += timedelta(days=1)
            for h in hours:
                if start <= h < return_t:
                    lbl = h.strftime("%H:%M")
                    if lbl in matrix.columns:
                        matrix.loc[vid, lbl] = 1

        fig = px.imshow(matrix, color_continuous_scale="Blues", title="Погрузка")
        fig.update_layout(height=300)
        st.plotly_chart(fig, use_container_width=True)

# === KPI ===
with col2:
    st.subheader("📈 KPI")
    df = st.session_state.result_df
    if not df.empty:
        total_waiting = df['Ожидание (мин)'].sum()
        total_trips = len(df)
        avg_waiting = total_waiting / total_trips
        total_time_in_trips = df['Время в рейсе (мин)'].sum() / 60  # в часах
        trips_by_type = df.groupby("Тип").size()

        st.metric("Всего рейсов", total_trips)
        st.metric("Суммарный простой (мин)", int(total_waiting))
        st.metric("Средний простой (мин)", f"{avg_waiting:.1f}")
        st.metric("Общее время в рейсах (ч)", f"{total_time_in_trips:.1f}")

        st.write("**Рейсов по типам:**")
        for t, cnt in trips_by_type.items():
            st.write(f"- {t}: {cnt}")

# === Gantt и Карта ===
st.subheader("📅 График рейсов")
if not st.session_state.result_df.empty:
    df = st.session_state.result_df
    gantt_data = []
    for _, row in df.iterrows():
        start_load = datetime.strptime(row['Подача'], "%H:%M")
        if start_load.hour < 10:
            start_load += timedelta(days=1)
        end_load = start_load + timedelta(minutes=30)
        arrive = datetime.strptime(row['Прибытие'], "%H:%M")
        if arrive.hour < 10:
            arrive += timedelta(days=1)
        depart = arrive + timedelta(minutes=row['Разгрузка до'] - row['Прибытие'])
        gantt_data.append(dict(Task=f"М{row['Машина']}", Start=start_load, Finish=end_load, Type="Погрузка"))
        gantt_data.append(dict(Task=f"М{row['Машина']}", Start=arrive, Finish=depart, Type="Разгрузка"))

    fig = go.Figure()
    for item in gantt_
        fig.add_trace(go.Bar(y=[item["Task"]], x=[(item["Finish"]-item["Start"]).seconds/3600],
                           base=item["Start"], orientation='h', marker_color="steelblue" if item["Type"]=="Погрузка" else "green",
                           showlegend=False))
    fig.update_layout(title="Gantt", height=400)
    st.plotly_chart(fig, use_container_width=True)

# Карта
if st.session_state.gmaps_api_key and not st.session_state.result_df.empty:
    st.subheader("🗺️ Маршруты")
    try:
        gmaps = googlemaps.Client(key=st.session_state.gmaps_api_key)
        base_url = "https://maps.googleapis.com/maps/api/staticmap?"
        markers = [f"markers=color:blue|{st.session_state.depot_address}"]
        path = ["path=weight:5|", st.session_state.depot_address]
        for _, row in st.session_state.result_df.iterrows():
            addr = st.session_state.store_addresses.get(row["Магазин"], "")
            if addr:
                markers.append(f"markers=color:red|{addr}")
                path.append(addr)
        url = base_url + "&".join(markers) + "&" + "&".join(path) + f"&size=600x300&key={st.session_state.gmaps_api_key}"
        st.image(url)
    except:
        st.info("Не удалось загрузить карту")