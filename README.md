import streamlit as st
import json
import os
import hashlib
import subprocess
from datetime import datetime

st.set_page_config(page_title="Pi Loco Tasks", layout="wide")

TASKS_FILE = "tasks.json"
IP_LOCO = "192.168.1.20"

# --- Utils ---
def hash_password(pwd):
    return hashlib.sha256(pwd.encode()).hexdigest()

def load_tasks():
    if not os.path.exists(TASKS_FILE):
        return []
    try:
        with open(TASKS_FILE, "r", encoding="utf-8") as f:
            return json.load(f)
    except:
        return []

def save_tasks(tasks):
    with open(TASKS_FILE, "w", encoding="utf-8") as f:
        json.dump(tasks, f, indent=2, ensure_ascii=False)

def ping_loco():
    try:
        result = subprocess.run(
            ["ping", "-c", "3", "-W", "1", IP_LOCO],
            capture_output=True, text=True, timeout=5
        )
        return result.returncode == 0, result.stdout
    except:
        return False, "Erreur ping"

# --- Login ---
if "logged_in" not in st.session_state:
    st.session_state.logged_in = False

if not st.session_state.logged_in:
    st.title("🔐 Connexion - Pi Loco")
    user = st.text_input("Utilisateur")
    pwd = st.text_input("Mot de passe", type="password")
    col1, col2 = st.columns(2)
    with col1:
        if st.button("Connexion Patron", use_container_width=True):
            if user == "patron" and hash_password(pwd) == hash_password("admin"):
                st.session_state.logged_in = True
                st.session_state.role = "patron"
                st.session_state.name = "Patron"
                st.rerun()
            else:
                st.error("Mauvais identifiants patron")
    with col2:
        if st.button("Connexion Personnel", use_container_width=True):
            if user == "personnel" and hash_password(pwd) == hash_password("1234"):
                st.session_state.logged_in = True
                st.session_state.role = "personnel"
                st.session_state.name = "Personnel"
                st.rerun()
            else:
                st.error("Mauvais identifiants personnel")
    st.stop()

# --- App ---
st.sidebar.write(f"Connecté: **{st.session_state.name}**")
if st.sidebar.button("Déconnexion"):
    st.session_state.logged_in = False
    st.rerun()

st.title("📡📋 Pi + Loco - Contrôle & Tâches")

# --- Section Ping Loco ---
st.subheader("Status Loco M5")
col1, col2, col3 = st.columns([1,1,2])
with col1:
    if st.button("Ping Loco", use_container_width=True):
        with st.spinner("Ping en cours..."):
            ok, output = ping_loco()
            if ok:
                st.session_state.last_ping = "OK"
                st.session_state.ping_time = datetime.now().strftime("%H:%M:%S")
            else:
                st.session_state.last_ping = "HS"
                st.session_state.ping_time = datetime.now().strftime("%H:%M:%S")
            st.session_state.ping_output = output
with col2:
    st.link_button("Ouvrir Loco", f"http://{IP_LOCO}", use_container_width=True)
with col3:
    if "last_ping" in st.session_state:
        if st.session_state.last_ping == "OK":
            st.success(f"Loco {IP_LOCO} OK à {st.session_state.ping_time}")
        else:
            st.error(f"Loco {IP_LOCO} HS à {st.session_state.ping_time}")

if "ping_output" in st.session_state:
    with st.expander("Voir détails ping"):
        st.code(st.session_state.ping_output)

st.divider()

# --- Section Tâches ---
if "tasks" not in st.session_state:
    st.session_state.tasks = load_tasks()

st.subheader("Suivi des tâches")

if st.session_state.role == "patron":
    with st.form("add_task", clear_on_submit=True):
        col1, col2 = st.columns([3,1])
        with col1:
            name = st.text_input("Nom de la tâche")
        with col2:
            freq = st.selectbox("Fréquence", ["Jour", "Semaine", "Mois"])
        if st.form_submit_button("Ajouter tâche") and name:
            st.session_state.tasks.append({
                "name": name,
                "frequency": freq,
                "done_me": False,
                "done_boss": False,
                "created": datetime.now().strftime("%d/%m %H:%M")
            })
            save_tasks(st.session_state.tasks)
            st.rerun()

# Compteur
en_cours = sum(1 for t in st.session_state.tasks if not (t["done_me"] and t["done_boss"]))
col1, col2, col3 = st.columns(3)
col1.metric("Total tâches", len(st.session_state.tasks))
col2.metric("En cours", en_cours)
col3.metric("Validées", len(st.session_state.tasks) - en_cours)

# Liste tâches
for i, t in enumerate(st.session_state.tasks):
    done = t["done_me"] and t["done_boss"]
    with st.container(border=True):
        col1, col2, col3, col4 = st.columns([4, 1, 1, 1])
        with col1:
            if done:
                st.markdown(f"~~**{t['name']}**~~")
            else:
                st.markdown(f"**{t['name']}**")
            st.caption(f"{t['frequency']} | Créée le {t['created']}")
        with col2:
            me = st.checkbox("Perso", value=t["done_me"], key=f"me{i}",
                           disabled=st.session_state.role=="patron")
            if me!= t["done_me"]:
                st.session_state.tasks[i]["done_me"] = me
                save_tasks(st.session_state.tasks)
                st.rerun()
        with col3:
            boss = st.checkbox("Patron", value=t["done_boss"], key=f"boss{i}",
                             disabled=st.session_state.role=="personnel")
            if boss!= t["done_boss"]:
                st.session_state.tasks[i]["done_boss"] = boss
                save_tasks(st.session_state.tasks)
                st.rerun()
        with col4:
            if done:
                st.markdown("✅ **Validé**")
            else:
                st.markdown("🔴 **À faire**")
            if st.session_state.role == "patron":
                if st.button("🗑️", key=f"del{i}", help="Supprimer"):
                    st.session_state.tasks.pop(i)
                    save_tasks(st.session_state.tasks)
                    st.rerun()
