import turtle
import time
import random
import paho.mqtt.client as mqtt

# Função de callback chamada quando o cliente recebe uma resposta CONNACK do servidor
def on_connect(client, userdata, flags, rc):
    print("Conectado ao HiveMQ com código de resultado: " + str(rc))

# Pontuação
pontuacao = 0
recorde = 0

# Inicializar o cliente MQTT
client = mqtt.Client(transport='websockets')

# Configurar as funções de callback
client.on_connect = on_connect

# Conectar ao HiveMQ usando WebSockets (substitua os detalhes do host e porta conforme necessário)
client.connect("mqtt-dashboard.com", 8884, 60)

# Configurar o tópico MQTT
topico_mqtt = "snake_game_dbsco"

atraso = 0.1

# Configurar a tela
wn = turtle.Screen()
wn.title("Snake Game")
wn.bgcolor("green")
wn.setup(width=600, height=600)
wn.tracer(0)  # Desativa as atualizações da tela

# Cabeça da cobra
cabeca = turtle.Turtle()
cabeca.speed(0)
cabeca.shape("square")
cabeca.color("black")
cabeca.penup()
cabeca.goto(0, 0)
cabeca.direction = "stop"

# Comida da cobra
comida = turtle.Turtle()
comida.speed(0)
comida.shape("circle")
comida.color("red")
comida.penup()
comida.goto(0, 100)

segmentos = []

# Corpo
corpo = turtle.Turtle()
corpo.speed(0)
corpo.shape("square")
corpo.color("white")
corpo.penup()
corpo.hideturtle()
corpo.goto(0, 260)
corpo.write("Pontuação: 0  Recorde: 0", align="center", font=("Courier", 24, "normal"))

# Funções
def ir_cima():
    if cabeca.direction != "baixo":
        cabeca.direction = "cima"

def ir_baixo():
    if cabeca.direction != "cima":
        cabeca.direction = "baixo"

def ir_esquerda():
    if cabeca.direction != "direita":
        cabeca.direction = "esquerda"

def ir_direita():
    if cabeca.direction != "esquerda":
        cabeca.direction = "direita"

def mover():
    if cabeca.direction == "cima":
        y = cabeca.ycor()
        cabeca.sety(y + 20)

    if cabeca.direction == "baixo":
        y = cabeca.ycor()
        cabeca.sety(y - 20)

    if cabeca.direction == "esquerda":
        x = cabeca.xcor()
        cabeca.setx(x - 20)

    if cabeca.direction == "direita":
        x = cabeca.xcor()
        cabeca.setx(x + 20)

# Associação de teclas
wn.listen()
wn.onkey(ir_cima, "Up")
wn.onkey(ir_baixo, "Down")
wn.onkey(ir_esquerda, "Left")
wn.onkey(ir_direita, "Right")

# Loop principal do jogo
while True:
    wn.update()

    # Verificar colisão com a borda
    if cabeca.xcor() > 290 or cabeca.xcor() < -290 or cabeca.ycor() > 290 or cabeca.ycor() < -290:
        time.sleep(1)
        cabeca.goto(0, 0)
        cabeca.direction = "stop"

        # Esconder os segmentos
        for segmento in segmentos:
            segmento.goto(1000, 1000)

        # Limpar a lista de segmentos
        segmentos.clear()

        # Resetar a pontuação
        pontuacao = 0

        # Resetar o atraso
        atraso = 0.1

        corpo.clear()
        corpo.write("Pontuação: {}  Recorde: {}".format(pontuacao, recorde), align="center", font=("Courier", 24, "normal"))

      
    # Verificar colisão com a comida
    if cabeca.distance(comida) < 20:
        # Mover a comida para um local aleatório
        x = random.randint(-290, 290)
        y = random.randint(-290, 290)
        comida.goto(x, y)

        # Adicionar um segmento
        novo_segmento = turtle.Turtle()
        novo_segmento.speed(0)
        novo_segmento.shape("square")
        novo_segmento.color("gray")
        novo_segmento.penup()
        segmentos.append(novo_segmento)

        # Diminuir o atraso
        atraso -= 0.001

        # Aumentar a pontuação
        pontuacao += 10

        if pontuacao > recorde:
            recorde = pontuacao

        corpo.clear()
        corpo.write("Pontuação: {}  Recorde: {}".format(pontuacao, recorde), align="center", font=("Courier", 24, "normal"))

        # Enviar pontuação para o tópico MQTT
        client.publish(topico_mqtt, pontuacao)

    # Mover os segmentos finais em ordem reversa
    for indice in range(len(segmentos) - 1, 0, -1):
        x = segmentos[indice - 1].xcor()
        y = segmentos[indice - 1].ycor()
        segmentos[indice].goto(x, y)

    # Mover o segmento 0 para onde a cabeça está
    if len(segmentos) > 0:
        x = cabeca.xcor()
        y = cabeca.ycor()
        segmentos[0].goto(x, y)

    mover()

    # Verificar colisão da cabeça com os segmentos do corpo
    for segmento in segmentos:
        if segmento.distance(cabeca) < 20:
            time.sleep(1)
            cabeca.goto(0, 0)
            cabeca.direction = "stop"

            # Esconder os segmentos
            for segmento in segmentos:
                segmento.goto(1000, 1000)

            # Limpar a lista de segmentos
            segmentos.clear()

            # Resetar a pontuação
            pontuacao = 0

            # Resetar o atraso
            atraso = 0.1

            # Atualizar a exibição da pontuação
            corpo.clear()
            corpo.write("Pontuação: {}  Recorde: {}".format(pontuacao, recorde), align="center", font=("Courier", 24, "normal"))

    time.sleep(atraso)

wn.mainloop()
