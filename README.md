# Graphwar-autocheat
Cheat pro grapwar (FEITO COM IA)
Como funciona
Visão: captura a tela (mss) e acha, sem precisar de calibração manual:
a janela do jogo e o campo (retângulo claro com proporção 770×450);
os dois times, agrupando os soldados por cor;
os obstáculos (tudo que não é fundo nem soldado e é "grosso", então linhas finas e texto são ignorados);
a caixa de texto onde se digita a função.
Planejamento: como a curva do Graphwar é uma função y = f(x), o caminho precisa ser sempre crescente em x. O script faz uma busca por programação dinâmica coluna a coluna, preferindo caminhos retos e evitando terreno e aliados. Ele tenta passar por todos os inimigos num único tiro e, se não houver caminho livre, tenta com menos alvos.
Função: o caminho (uma poligonal) vira uma expressão usando abs():
   f(x) = m0*x + Σ (Δm_i / 2) * ( |x - x_i| + x - x_i )

em que m0 é a inclinação inicial e Δm_i a mudança de inclinação em cada ponto de dobra x_i. Considera-se x = 0 no seu soldado e um campo de 50 × 30 unidades. 4. Automação: espera a tela ficar parada (~1 s), calcula, clica na caixa, digita a função e aperta Enter. Depois espera a animação do tiro e repete.
Python 3.9 ou mais novo (testado com 3.14)
Windows (a detecção da janela e o ajuste de zoom da tela usam recursos do Windows)
Graphwar aberto 
Quando pedir, deixe o mouse em cima do seu soldado por 4 segundos (é só pra ele saber qual dos dois times é o seu).
Ele cria o arquivo deteccao.png mostrando o que enxergou: campo em verde, seus soldados em verde, inimigos em vermelho e a caixa de função com uma cruz rosa. Confira se bate com o jogo.
Pra parar: Ctrl+C no terminal ou leve o mouse a um canto da tela (fail-safe do pyautogui).

Pra só ver as funções sem atirar, mude SHOOT = False (em graphwar_mini.py) ou DISPARAR = False (em graphwar_auto.py).

Configuração (versão completa)

No topo do graphwar_auto.py, tudo começa em None (automático). Só preencha se a detecção falhar:

Opção	O que é
REGIAO	{"left":..,"top":..,"width":..,"height":..} do campo, em pixels da tela
CAMPO_FUNCAO	(x, y) da caixa onde se digita a função
COR_TIME / COR_INIMIGO	Cores (R, G, B) dos dois times
COR_ATIVO	Cor do marcador do soldado da vez, se o jogo tiver um
PASSO, MAX_DY	Resolução e inclinação máxima do caminho (aumente se a função ficar longa demais)


CODIGO:
"""Graphwar automático (versão compacta).  pip install numpy scipy mss pyautogui pillow
Abra uma partida LOCAL e rode: python graphwar_mini.py   (parar: Ctrl+C ou mouse num canto da tela)"""
import time
import numpy as np
from scipy import ndimage as nd
try:
    import ctypes; ctypes.windll.shcore.SetProcessDpiAwareness(2)   # evita desencontro de zoom no Windows
except Exception:
    pass

SHOOT = True            # False = só mostra a função, sem digitar/atirar
PASSO, MAXDY = 6, 30    # resolução do caminho (px)
CFG = {}

cap = lambda s, r: np.array(s.grab(r))[:, :, [2, 1, 0]]                  # BGRA -> RGB
colorido = lambda im: (im.max(2).astype(int) - im.min(2)) > 80           # soldados (não cinza/branco/preto)

def blobs(im):
    m = colorido(im); lab, n = nd.label(m)
    if not n: return []
    i = range(1, n + 1)
    K = np.stack([nd.mean(im[:, :, c], lab, i) for c in range(3)], 1)
    return [(x, y, k) for (y, x), a, k in zip(nd.center_of_mass(m, lab, i), nd.sum(m, lab, i), K) if 40 <= a <= 4000]

def times(im):                                   # agrupa soldados por cor; os 2 maiores grupos são os times
    g = []
    for x, y, k in blobs(im):
        for t in g:
            if abs(t[0] - k).sum() < 120: t[1].append((x, y)); break
        else: g.append([k, [(x, y)]])
    return sorted(g, key=lambda t: -len(t[1]))[:2]

def ler(im):
    meus, ini = [], []
    for x, y, k in blobs(im):
        dt, di = abs(CFG["meu"] - k).sum(), abs(CFG["ini"] - k).sum()
        if min(dt, di) < 120: (meus if dt < di else ini).append((x, y))
    return meus, ini

def retangulos(im):                              # caixas (x, y, w, h) das regiões brancas
    lab, _ = nd.label(nd.binary_closing(im.min(2) >= 248, np.ones((3, 3))))
    return [(s[1].start, s[0].start, s[1].stop - s[1].start, s[0].stop - s[0].start) for s in nd.find_objects(lab)]

def apontar(msg):
    import pyautogui
    for s in range(4, 0, -1): print(f"{msg} ({s}s)", end="\r", flush=True); time.sleep(1)
    return tuple(pyautogui.position())

def janela():
    try:
        import pygetwindow as gw
        for j in gw.getWindowsWithTitle("Graphwar"):
            if j.width > 300 and j.height > 300:
                try: j.isMinimized and j.restore(); j.activate()
                except Exception: pass
                time.sleep(.8); return j.left, j.top, j.width, j.height
    except Exception: pass

def preparar():
    import mss, pyautogui
    from PIL import Image, ImageDraw
    pyautogui.FAILSAFE = True
    jan = janela() or print("Aviso: janela 'Graphwar' não achada; deixe o jogo visível.")
    with mss.mss() as s:
        mon = s.monitors[1]; ox, oy = mon["left"], mon["top"]; tela = cap(s, mon)
        bx, by = (max(jan[0] - ox, 0), max(jan[1] - oy, 0)) if jan else (0, 0)
        sub = tela[by:by + jan[3], bx:bx + jan[2]] if jan else tela
        R = retangulos(sub)
        C = [r for r in R if r[2] >= 400 and r[3] >= 230 and abs(r[2] / r[3] - 770 / 450) < .08]
        if C:
            fx, fy, w, h = max(C, key=lambda r: r[2] * r[3]); x, y = ox + bx + fx, oy + by + fy
        else:
            print("Não achei o campo sozinho:")
            (x, y), (x2, y2) = apontar("Mouse no canto SUPERIOR ESQUERDO do campo"), apontar("Mouse no canto INFERIOR DIREITO")
            w, h, fx, fy = x2 - x, y2 - y, x - ox - bx, y - oy - by
        CFG["reg"] = reg = dict(left=x, top=y, width=w, height=h); im = cap(s, reg)
        v, n = np.unique(im[::3, ::3].reshape(-1, 3) // 8, axis=0, return_counts=True); CFG["fundo"] = v[n.argmax()] * 8 + 4
        g = times(im)
        if len(g) < 2: raise RuntimeError("Não achei os dois times. Abra uma partida e rode de novo.")
        mx, my = apontar("Deixe o mouse em cima do SEU soldado")
        d, k = min((np.hypot(px - (mx - x), py - (my - y)), i) for i, t in enumerate(g) for px, py in t[1])
        if d > 60: k = min((0, 1), key=lambda i: np.mean([p[0] for p in g[i][1]]))   # sem mouse: time da esquerda
        CFG["meu"], CFG["ini"] = g[k][0], g[1 - k][0]
        B = [r for r in R if 14 <= r[3] <= 50 and r[2] >= 120 and r[2] / r[3] >= 4 and not (fx <= r[0] <= fx + w and fy <= r[1] <= fy + h)]
        if B:
            r = min(B, key=lambda r: (r[1] < fy + h, abs(r[1] - fy - h), -r[2]))
            CFG["caixa"] = (ox + bx + r[0] + r[2] // 2, oy + by + r[1] + r[3] // 2)
        else: CFG["caixa"] = apontar("Deixe o mouse em cima da CAIXA onde se digita a função")
        meus, ini = ler(im)
        pic = Image.fromarray(tela); dr = ImageDraw.Draw(pic); L, T = x - ox, y - oy      # imagem de conferência
        dr.rectangle([L, T, L + w, T + h], outline=(0, 200, 0), width=3)
        for pts, c in ((meus, (0, 200, 0)), (ini, (255, 0, 0))):
            for px, py in pts: dr.ellipse([L + px - 14, T + py - 14, L + px + 14, T + py + 14], outline=c, width=3)
        cx, cy = CFG["caixa"][0] - ox, CFG["caixa"][1] - oy
        dr.line([cx - 15, cy, cx + 15, cy], fill=(255, 0, 255), width=3); dr.line([cx, cy - 15, cx, cy + 15], fill=(255, 0, 255), width=3)
        pic.save("deteccao.png")
    print(f"Campo {w}x{h} | seu time: {len(meus)} | inimigos: {len(ini)} | caixa: {CFG['caixa']} | veja deteccao.png")

def livre(b, p, q):
    n = int(max(abs(q[0] - p[0]), abs(q[1] - p[1]))) + 1
    return not b[np.linspace(p[1], q[1], n).astype(int).clip(0, b.shape[0] - 1),
                 np.linspace(p[0], q[0], n).astype(int).clip(0, b.shape[1] - 1)].any()

def trecho(b, a, c):                             # caminho x-crescente (uma função!) de a até c, sem bater em nada
    xs = list(range(int(a[0]), int(c[0]), PASSO)) + [int(c[0])]
    camada, pais = {int(a[1]): 0}, [{}]
    for i in range(1, len(xs)):
        novo, pai = {}, {}
        for y1, k in camada.items():
            for y2 in [int(c[1])] if i == len(xs) - 1 else range(y1 - MAXDY, y1 + MAXDY + 1, PASSO):
                if 0 <= y2 < b.shape[0] and abs(y2 - y1) <= MAXDY and livre(b, (xs[i - 1], y1), (xs[i], y2)):
                    cc = k + (y2 - y1) ** 2 + 1                  # prefere caminhos retos
                    if cc < novo.get(y2, 1e18): novo[y2], pai[y2] = cc, y1
        if not novo: return None
        camada = novo; pais.append(pai)
    y, out = int(c[1]), [(xs[-1], int(c[1]))]
    for i in range(len(xs) - 1, 0, -1): y = pais[i][y]; out.append((xs[i - 1], y))
    return out[::-1]

def caminho(b, o, alvos):
    P = [o]
    for a, c in zip([o] + alvos, alvos):
        t = trecho(b, a, c)
        if t is None: return None
        P += t[1:]
    return P

def expr(P, o, w, h):                            # poligonal -> f(x) com abs(); x=0 é o seu soldado
    q = [((x - o[0]) / (w / 50), (o[1] - y) / (h / 30)) for x, y in P]
    q = [p for i, p in enumerate(q) if i == 0 or p[0] > q[i - 1][0]]
    m = [(b[1] - a[1]) / (b[0] - a[0]) for a, b in zip(q, q[1:])]
    return f"{m[0]:.4f}*x" + "".join(f"+({(n - p) / 2:.4f})*(abs(x-{x:.3f})+x-{x:.3f})"
                                    for (x, _), p, n in zip(q[1:], m, m[1:]) if abs(n - p) > 1e-3)

def calcular(im, ref):
    h, w = im.shape[:2]; meus, ini = ler(im)
    if not meus or not ini: raise RuntimeError("Não achei soldados nos dois times (fim de partida?).")
    eu = min(meus, key=lambda c: (c[0] - ref[0]) ** 2 + (c[1] - ref[1]) ** 2)
    al = [c for c in meus if c is not eu]
    solido = (np.abs(im.astype(int) - CFG["fundo"]).sum(2) > 60) & ~colorido(im)    # terreno: grosso, não é fundo nem soldado
    b = nd.binary_dilation(nd.binary_opening(solido, np.ones((5, 5))), iterations=3)
    if min(ini, key=lambda c: abs(c[0] - eu[0]))[0] < eu[0]:                          # inimigo à esquerda: espelha tudo
        f = lambda c: (w - 1 - c[0], c[1]); b = b[:, ::-1]; eu, al, ini = f(eu), [f(c) for c in al], [f(c) for c in ini]
    for x, y in al: b[max(0, int(y) - 12):int(y) + 12, max(0, int(x) - 12):int(x) + 12] = True
    for x, y in [eu] + ini: b[max(0, int(y) - 8):int(y) + 8, max(0, int(x) - 8):int(x) + 8] = False
    alvos = []
    for c in sorted((c for c in ini if c[0] > eu[0] + PASSO), key=lambda c: c[0]):
        if not alvos or c[0] > alvos[-1][0] + PASSO: alvos.append(c)
    for k in range(len(alvos), 0, -1):           # tenta pegar todos de uma vez; se não der, menos alvos
        P = caminho(b, eu, alvos[:k])
        if P: return expr(P, eu, w, h), k
    return None, 0

def jogar():
    import mss, pyautogui
    ant, parado, espera, t0 = None, 0, False, 0
    print("Modo automático ligado.")
    with mss.mss() as s:
        while True:
            time.sleep(.25); im = cap(s, CFG["reg"]); a = im.astype(int)
            mudou = ant is None or abs(a - ant).mean() > .5; ant = a
            if mudou: parado, espera = 0, False; continue                    # animação rolando
            parado += 1
            if espera and time.time() - t0 > 8: espera = False               # tiro não saiu; tenta de novo
            if parado < 4 or espera: continue                                # ~1s de tela parada = nossa vez
            meus, _ = ler(im)
            if not meus: continue
            ref = meus[0] if len(meus) == 1 else (pyautogui.position()[0] - CFG["reg"]["left"], pyautogui.position()[1] - CFG["reg"]["top"])
            try: e, k = calcular(im, ref)
            except RuntimeError as err: print(err); parado = 0; continue
            if not e: print("Nenhum caminho livre."); parado = 0; continue
            print(f"Acerta {k} inimigo(s): y = {e}")
            if SHOOT:
                pyautogui.click(CFG["caixa"]); pyautogui.hotkey("ctrl", "a"); pyautogui.typewrite(e, interval=.01); pyautogui.press("enter")
            espera, t0, parado = True, time.time(), 0

if __name__ == "__main__":
    preparar(); jogar()
    
