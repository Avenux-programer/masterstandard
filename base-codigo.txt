from dash import html, dcc, Input, Output, dash_table, callback_context
import dash_auth
import pandas as pd
import locale
import os
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import dash 
from babel.numbers import format_currency
from dash import Dash 
from dash.dependencies import Input, Output 
import flask



app = Dash(__name__) # Criação do aplicativo Dash
server = app.server # Cria uma instância do servidor WSGI (Gunicorn espera isso)

# Definição de dicionários de usuários
USUARIOS = {
    "CRISTIAN": os.environ.get('CRISTIAN_USER', 'redentor'),
    "RAFAEL": os.environ.get('RAFAEL_USER', '502020'),
    "ADRIANA": os.environ.get('ADRIANA_USER', '51204'),
    "RODRIGO": os.environ.get('RODRIGO_USER', '502020'),
    "FELIPE": os.environ.get('FELIPE_USER', '502020'),
    "SERGIO": os.environ.get('SERGIO_USER', '51204'),
    "DARCIONE": os.environ.get('DARCIONE_USER', '502020'),
    "THALES": os.environ.get('THALES_USER', '502020'),
    "WESLEY": os.environ.get('WESLEY_USER', '502020'),
    # ...
}

auth = dash_auth.BasicAuth(app, USUARIOS)

# Função para atualizar os dados do DataFrame quando o arquivo Excel for modificado
def update_data():
    global df
    df = pd.read_excel("teste_standard.xlsx")
    df['DATA'] = df['DATA'].dt.strftime('%d/%m/%Y')
    df['DESCRICAO'] = df['DESCRICAO']
    df['CODIGOFILHO'] = df['CODIGOFILHO']

    try:
        locale.setlocale(locale.LC_ALL, 'pt_BR.UTF-8')
    except locale.Error:
        locale.setlocale(locale.LC_ALL, 'C')

    def formatar_numero(numero):
        return locale.format_string('%.2f', numero, grouping=True)

    df['REALIZADO'] = df['REALIZADO'].apply(formatar_numero)
    df['PREVISTO'] = df['PREVISTO'].apply(formatar_numero)
    df['DIFERENCA'] = df['DIFERENCA'].apply(formatar_numero)

    try:
        locale.setlocale(locale.LC_ALL, 'pt_BR.UTF-8')
    except locale.Error:
        locale.setlocale(locale.LC_ALL, 'C')

    # Verificar se a coluna 'VALORDIF' existe no DataFrame
    if 'VALORDIF' in df.columns:
        df['VALORDIF'] = df['VALORDIF'].apply(lambda x: format_currency(x, 'BRL', locale='pt_BR'))
    else:
        print("A coluna 'VALORDIF' não foi encontrada no DataFrame.")

# Inicializa o DataFrame e atualiza os dados
df = pd.read_excel("teste_standard.xlsx")
update_data()

lista_data = list(df["DATA"].unique())
lista_data.append("Todas as Datas")

# Use a coluna "DIFERENCA" em vez de "VALORDIF" para as opções do filtro
lista_diferenca = list(df["DIFERENCA"].unique())
lista_diferenca.append("Todas as Diferenças")

lista_descricao = list(df["DESCRICAO"].unique())
lista_descricao.append("Todas as Descrições")

lista_codigofilho = list(df["CODIGOFILHO"].unique())
lista_codigofilho.append("Todos os Códigos Filho")

# Função para retornar o DataFrame completo
def get_full_dataframe():
    return df.to_dict("records") #possível erro ta aqui ---------

# Layout do aplicativo
app.layout = html.Div(children=[
    html.H1(children='Master Industria'),

    html.Div(children='''
        
    '''),

    html.Div(children=[
        html.Label("Filtrar por Data:"),
        dcc.Dropdown(options=[{'label': DATA, 'value': DATA} for DATA in lista_data], value=[], id='lista_data', style={"width": "50%", "margin": "auto"}, multi=True),
        html.Label("Filtrar por Diferença:"),
        dcc.Dropdown(options=[{'label': DIFERENCA, 'value': DIFERENCA} for DIFERENCA in lista_diferenca], value=[], id='lista_diferenca', style={"width": "50%", "margin": "auto"}, multi=True),
        html.Label("Filtrar por Descrição:"),
        dcc.Dropdown(options=[{'label': descricao, 'value': descricao} for descricao in lista_descricao], value=[], id='lista_descricao', style={"width": "50%", "margin": "auto"}, multi=True),
        html.Label("Filtrar por Código Filho:"),
        dcc.Dropdown(options=[{'label': codigofilho, 'value': codigofilho} for codigofilho in lista_codigofilho], value=[], id='lista_codigofilho', style={"width": "50%", "margin": "auto"}, multi=True),
        
    ], style={"display": "flex", "flex-direction": "column", "align-items": "center"}),

    html.H3(children='Dashboard Real vs Standard', id="subtitulo"),

    html.Div([
        html.Button('Redefinir Filtros', id='botao_reset'),
    ], style={"margin": "10px"}),  # Botão para redefinir os filtros

    html.Div([
        dash_table.DataTable(
            id='tabela_dados',
            columns=[{"name": col, "id": col} for col in df.columns],
            data=get_full_dataframe(),  # Inicializa com o DataFrame completo
            style_table={'overflowX': 'scroll'},
            style_cell={'whiteSpace': 'normal', 'height': 'auto', 'textAlign': 'center'},
            fixed_rows={'headers': True, 'data': 0}  # Define o cabeçalho como fixo
        )
    ], style={"overflowY": "scroll", "maxHeight": "500px"})  # Define a altura máxima da tabela com barra de rolagem

], style={"text-align": "center"})

# Configura o monitoramento do arquivo Excel
class ExcelFileEventHandler(FileSystemEventHandler):
    def on_modified(self, event):
        if event.src_path == "teste_standard.xlsx":
            update_data()

observer = Observer()
observer.schedule(ExcelFileEventHandler(), ".", recursive=False)
observer.start()

# Rota para redirecionar automaticamente para o link Render
@app.server.route("/")
def redirect_to_render():
    return flask.redirect("https://masterstandard.onrender.com")

# callbacks -> dar funcionalidade para o nosso dashboard (conecta os botões com os gráficos)
@app.callback(
    Output('tabela_dados', 'data'),  # Atualiza os dados da tabela,
    Input('lista_data', 'value'),
    Input('lista_diferenca', 'value'),
    Input('lista_descricao', 'value'),  # Novo input para a coluna "DESCRICAO"
    Input('lista_codigofilho', 'value'),  # Novo input para a coluna "CODIGOFILHO"
    Input('botao_reset', 'n_clicks')  # Monitor

)
def atualizar_tabela(DATA, DIFERENCA, DESCRICAO, CODIGOFILHO, n_clicks):
    print("Valores dos filtros:")
    print("DATA:", DATA)
    print("DIFERENCA:", DIFERENCA)

    ctx = dash.callback_context
    if not ctx.triggered:  # Nenhum callback foi acionado, retorna o DataFrame completo
        print("Nenhum callback acionado. Retornando o DataFrame completo.")
        return get_full_dataframe()

    prop_id = ctx.triggered[0]['prop_id']
    if prop_id == 'botao_reset.n_clicks':  # O botão de redefinir foi clicado
        print("Botão de redefinir foi clicado. Retornando o DataFrame completo.")
        return get_full_dataframe()

    df_filtrada = df

    if DATA:
        if "Todas as Datas" not in DATA:
            df_filtrada = df_filtrada.loc[df_filtrada['DATA'].isin(DATA), :]

    if DIFERENCA:
        if "Todas as Diferenças" not in DIFERENCA:
            df_filtrada = df_filtrada.loc[df_filtrada['DIFERENCA'].isin(DIFERENCA), :]
            
    if DESCRICAO:
        if "Todas as Descrições" not in DESCRICAO:
            df_filtrada = df_filtrada.loc[df_filtrada['DESCRICAO'].isin(DESCRICAO), :]

    if CODIGOFILHO:
        if "Todos os Códigos Filho" not in CODIGOFILHO:
            df_filtrada = df_filtrada.loc[df_filtrada['CODIGOFILHO'].isin(CODIGOFILHO), :]
            
    

    print("DataFrame filtrado:")
    print(df_filtrada)

    return df_filtrada.to_dict("records")

if __name__ == "__main__":
    app.run_server(host="0.0.0.0", port=8050, debug=False)
