# App.apy
SIGA estatísticas
import streamlit as st
import pandas as pd
import plotly.express as px
import io
import plotly.io as pio

st.set_page_config(page_title="SIGA Comparação", layout="wide")
st.title("SIGA: Dia Atual vs Mês Anterior")

# Upload
cols = st.columns(2)
with cols[0]:
    st.subheader("Dia Atual")
    espera_atual = st.file_uploader("Espera intervalos", type="xlsx", key="ea")
    tma_atual    = st.file_uploader("TMA/TME", type="xlsx", key="ta")
with cols[1]:
    st.subheader("Mês Anterior")
    espera_mes = st.file_uploader("Espera intervalos", type="xlsx", key="em")
    tma_mes    = st.file_uploader("TMA/TME", type="xlsx", key="tm")

if espera_atual and espera_mes and tma_atual and tma_mes:
    try:
        servicos = ["Geral", "Tesouraria", "Posto de serviços rápidos", "Marcação"]
        df_ea = pd.read_excel(espera_atual, skiprows=1).query("descricao in @servicos")
        df_em = pd.read_excel(espera_mes,   skiprows=1).query("descricao in @servicos")
        df_ta = pd.read_excel(tma_atual,    skiprows=1).query("descricao in @servicos")
        df_tm = pd.read_excel(tma_mes,      skiprows=1).query("descricao in @servicos")

        espera_cols = ["espera_0_5", "espera_5_10", "espera_10_20", "espera_20_30", "espera_30"]
        for df in [df_ea, df_em]:
            for c in espera_cols:
                df[c] = pd.to_numeric(df[c], errors='coerce').fillna(0)

        def to_min(v): return round(float(v) * 60, 1) if pd.notna(v) else 0
        for df in [df_ta, df_tm]:
            for c in ["TMA", "TME", "t_maxAtendimento", "t_maxEspera"]:
                if c in df.columns:
                    df[c] = df[c].apply(to_min)

        def pct_30(df):
            total = df[espera_cols].sum(axis=1)
            return (df["espera_30"] / total * 100).round(1).where(total > 0, 0)

        pct_a = pct_30(df_ea)
        pct_m = pct_30(df_em)
        global_a = round(df_ea["espera_30"].sum() / df_ea[espera_cols].sum().sum() * 100, 1)
        global_m = round(df_em["espera_30"].sum() / df_em[espera_cols].sum().sum() * 100, 1)

        def taxa_desist(df):
            t = pd.to_numeric(df["total"], errors='coerce').fillna(0)
            d = pd.to_numeric(df["encerrada_desistencia"], errors='coerce').fillna(0)
            return (d / t * 100).round(1).where(t > 0, 0)

        des_a = taxa_desist(df_ta)
        des_m = taxa_desist(df_tm)
        g_des_a = round(df_ta["encerrada_desistencia"].sum() / df_ta["total"].sum() * 100, 1) if df_ta["total"].sum() > 0 else 0
        g_des_m = round(df_tm["encerrada_desistencia"].sum() / df_tm["total"].sum() * 100, 1) if df_tm["total"].sum() > 0 else 0

        st.subheader("KPIs Principais")
        k1, k2, k3, k4 = st.columns(4)
        k1.metric("% >30 min", f"{global_a}%")
        k2.metric("vs mês ant.", f"{global_m}%")
        k3.metric("Taxa desist.", f"{g_des_a}%")
        k4.metric("vs mês ant.", f"{g_des_m}%")

        comp_pct = pd.DataFrame({
            "Serviço": df_ea["descricao"],
            "% >30 Hoje": pct_a,
            "% >30 Mês ant": pct_m,
            "Δ pp": (pct_a - pct_m).round(1)
        }).set_index("Serviço")

        comp_des = pd.DataFrame({
            "Serviço": df_ta["descricao"],
            "Desist. Hoje (%)": des_a,
            "Desist. Mês ant (%)": des_m,
            "Δ %": (des_a - des_m).round(1)
        }).set_index("Serviço")

        st.subheader("Comparação")
        st.dataframe(comp_pct.style.format("{:.1f}"), use_container_width=True)
        st.dataframe(comp_des.style.format("{:.1f}"), use_container_width=True)

        st.subheader("Gráficos")
        fig1 = px.bar(
            comp_pct.reset_index().melt(id_vars="Serviço", value_vars=["% >30 Hoje", "% >30 Mês ant"]),
            x="Serviço", y="value", color="variable", barmode="group",
            title="% Espera > 30 min"
        )
        st.plotly_chart(fig1, use_container_width=True)

        fig2 = px.bar(
            comp_des.reset_index().melt(id_vars="Serviço", value_vars=["Desist. Hoje (%)", "Desist. Mês ant (%)"]),
            x="Serviço", y="value", color="variable", barmode="group",
            title="Taxa de Desistência (%)"
        )
        st.plotly_chart(fig2, use_container_width=True)

        st.subheader("Exportar")
        for fig, nome in [(fig1, "espera_30"), (fig2, "desistencia")]:
            buf = io.BytesIO()
            try:
                pio.write_image(fig, buf, format="pdf", engine="kaleido")
                buf.seek(0)
                st.download_button(f"↓ Gráfico {nome} (PDF)", buf, f"{nome}_comparacao.pdf", "application/pdf")
            except:
                st.caption(f"Gráfico {nome}: exportação PDF não disponível (instale kaleido?)")

        st.info("Para PDF completo: Ctrl+P → Guardar como PDF")

    except Exception as e:
        st.error(f"Erro: {str(e)[:120]}...")

else:
    st.info("Carregue os quatro ficheiros para começar.")
