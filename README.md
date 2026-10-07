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
