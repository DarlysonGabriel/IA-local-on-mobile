# IA Local no Android com Termux

«_Execute uma IA localmente no seu celular Android, sem depender de APIs externas, utilizando Termux + llama.cpp + modelos GGUF._»

Este projeto mostra como transformar um smartphone Android em um pequeno assistente de IA local, utilizando o processador do próprio aparelho para executar o modelo.

O guia utiliza o SmolLM2 1.7B Instruct Q4_K_M como exemplo, mas outros modelos compatíveis com "llama.cpp" também podem ser utilizados.

---

# Sobre o projeto

A arquitetura utilizada é simples:
```
┌─────────────────────────────┐
│          Android            │
│                             │
│  ┌───────────────────────┐  │
│  │       Termux          │  │
│  │                       │  │
│  │  ┌─────────────────┐  │  │
│  │  │    llama.cpp    │  │  │
│  │  │                 │  │  │
│  │  │  llama-server   │  │  │
│  │  └────────┬────────┘  │  │
│  │           │           │  │
│  │  ┌────────▼────────┐  │  │
│  │  │    GGUF Model   │  │  │
│  │  │   SmolLM2 1.7B  │  │  │
│  │  └─────────────────┘  │  │
│  └───────────────────────┘  │
└──────────────┬──────────────┘
               │
               ▼
      http://192.x.x.1:8080
```
O "llama.cpp" funciona como o motor de inferência, enquanto o "llama-server" disponibiliza uma interface web local.

---

# Recursos

- Execução de IA local
- Funciona diretamente no Android
- Utiliza a CPU do smartphone
- Interface web local
- Não depende de uma API de IA externa
- Suporte a modelos quantizados em GGUF
- Chat Template personalizado
- Controle de temperatura, contexto, threads e batch
- Ambiente Linux através do Termux
- Pode ser utilizado em dispositivos com hardware limitado

---

## Requisitos

Requisitos mínimos

|Componente| Mínimo |
|----|-----|
|Sistema| Android 7+
Terminal| Termux
Arquitetura| ARM64 / "aarch64"
RAM| 4 GB
Armazenamento livre| ~3 GB
Internet| Necessária durante a instalação
CPU| ARM64

«O desempenho varia bastante de acordo com o processador e o gerenciamento térmico do aparelho.»

# Recomendado

Componente| Recomendado
|----|-----|
RAM| 6–8 GB
CPU| 6+ núcleos
Armazenamento livre| 4+ GB
Arquitetura| ARM64
Android| Versão relativamente recente

---

# 1. Instale o Termux

Instale uma versão atual do Termux.

Depois abra o aplicativo e atualize os pacotes:

``` pkg update && pkg upgrade -y ```

---

# 2. Instale as dependências

Instale as ferramentas necessárias para compilar o "llama.cpp":

``` pkg install git cmake clang make -y ```

# Verifique a arquitetura:

``` uname -m ```

O resultado esperado é:

``` aarch64 ```

Também podemos verificar quantos processadores estão disponíveis:

``` nproc ```

---

# 3. Baixe o llama.cpp

Clone o projeto:

``` git clone https://github.com/ggml-org/llama.cpp.git ```

Entre na pasta:

``` cd llama.cpp ```

---

# 4. Compile o llama.cpp

Configure o projeto:

``` cmake -B build ```

Compile:

``` cmake --build build --config Release -j$(nproc) ```

Essa etapa pode demorar dependendo do smartphone.

Teste o "llama-cli":

``` ./build/bin/llama-cli --help ```

E teste o servidor:

``` ./build/bin/llama-server --help ```

Se aparecer a tela de ajuda, o "llama.cpp" foi compilado corretamente.

---

# 5. Baixe o modelo

Crie uma pasta para os modelos:

``` mkdir -p ~/models ```

Este tutorial utiliza:

SmolLM2 1.7B Instruct — Q4_K_M

Baixe:
```
wget -O ~/models/smollm2-1.7b-instruct-q4_k_m.gguf \
https://huggingface.co/HuggingFaceTB/SmolLM2-1.7B-Instruct-GGUF/resolve/main/smollm2-1.7b-instruct-q4_k_m.gguf
```
Verifique:

``` ls -lh ~/models/ ```

Você deverá encontrar:

``` smollm2-1.7b-instruct-q4_k_m.gguf ```

O arquivo possui aproximadamente 1 GB.

---

# 6. Teste a IA pelo terminal

Entre na pasta do "llama.cpp":

``` cd ~/llama.cpp ```

Execute:
```
./build/bin/llama-cli \
-m ~/models/smollm2-1.7b-instruct-q4_k_m.gguf \
-c 2048 \
-n 256
```
Se o modelo responder, a inferência local está funcionando.

---

# 7. Execute o llama-server

Agora vamos utilizar a interface web:
```
~/llama.cpp/build/bin/llama-server \
-m ~/models/smollm2-1.7b-instruct-q4_k_m.gguf \
-c 2048 \
-t 6 \
-b 256 \
--host {seu_ip} \
--port 8080
```
Abra o navegador do Android e acesse:

``` http://{seu_ip}:8080 ```

Você deverá encontrar a interface web do "llama-server".

---

# 8. Chat Template personalizado

O modelo utiliza um Chat Template para transformar as mensagens da conversa em um formato que ele consegue interpretar.

O "llama-server" permite carregar um template Jinja através de um arquivo externo.

Crie o arquivo:

``` nano ~/smollm-template.jinja ```

Cole:
```
{% for message in messages %}
{% if loop.first and messages[0]['role'] != 'system' %}
{{ '<|im_start|>system
You are a local AI programming mentor.

Respond in Portuguese unless the user asks for another language.

Help with Linux, Git, GitHub, Godot, game development, programming, and general technology.

Be precise, practical, and concise. Do not blindly agree with the user. If the user is incorrect, explain what is wrong and why.

Prefer lightweight solutions suitable for low-resource hardware.

When providing code, keep changes focused and avoid unnecessarily rewriting the user''s project.
<|im_end|>
' }}
{% endif %}
{{ '<|im_start|>' + message['role'] + '
' + message['content'] + '<|im_end|>
' }}
{% endfor %}
{% if add_generation_prompt %}
{{ '<|im_start|>assistant
' }}
{% endif %}
```
Esse prompt transforma o modelo em um assistente focado em:

- Linux
- Programação
- Godot
- Desenvolvimento de jogos
- Git
- GitHub
- Tecnologia

O conteúdo do "system" pode ser alterado para criar diferentes tipos de assistentes.

---

# 9. Crie um script para iniciar a IA

Para não precisar digitar todos os parâmetros toda vez, crie:

``` nano ~/start-ai.sh ```

Coloque:
```
#!/data/data/com.termux/files/usr/bin/bash

cd ~/llama.cpp

./build/bin/llama-server \
  -m ~/models/smollm2-1.7b-instruct-q4_k_m.gguf \
  -c 2048 \
  -t 6 \
  -b 256 \
  --host {seu_ip} \
  --port 8080 \
  --temp 0.6 \
  --top-p 0.9 \
  --repeat-penalty 1.1 \
  --chat-template-file ~/smollm-template.jinja
```
---

# 10. Dê permissão ao script

Execute:

``` chmod +x ~/start-ai.sh ```

Agora você pode iniciar o servidor usando:

``` ~/start-ai.sh ```

---

# Configuração de desempenho

Uma configuração utilizada neste projeto:

Configuração| Valor
|----|-----|
Contexto| "2048"
Threads| "6"
Batch| "256"
Temperature| "0.6"
Top-P| "0.9"
Repeat penalty| "1.1"

## Threads

Você pode descobrir a quantidade de CPUs disponíveis:

``` nproc ```

Depois pode testar diferentes valores:
```
-t 4
-t 6
-t 8
```
O maior número de threads não necessariamente produz maior velocidade.

Faça testes no próprio aparelho.

---

## Contexto

O contexto controla aproximadamente quanto conteúdo anterior pode permanecer disponível para o modelo.

Exemplo:
```
-c 1024
```
ou:
```
-c 2048
```
Um contexto maior pode manter mais informações da conversa, mas também aumenta o consumo de memória e o trabalho de processamento.

---

## Desempenho

A velocidade pode ser observada em:
```
tokens/s
```
Por exemplo:
```
3.30 t/s
```
significa aproximadamente 3,3 tokens gerados por segundo.

O desempenho depende de:

- CPU
- Número de threads
- Frequência do processador
- Temperatura
- Thermal throttling
- RAM disponível
- Tamanho do modelo
- Quantização
- Contexto
- Batch size

---

## Temperatura

A inferência de modelos pode utilizar bastante CPU durante períodos prolongados.

Se o aparelho aquecer, o Android pode reduzir automaticamente a frequência da CPU.

Isso é chamado de thermal throttling.

Não é recomendado desativar mecanismos de proteção térmica ou tentar fazer overclock.

Para sessões longas, mantenha o aparelho em uma superfície ventilada e evite deixá-lo abafado.

---

# Estrutura final

Depois de completar o tutorial:

```
├── llama.cpp/
│   └── build/
│       └── bin/
│           ├── llama-cli
│           └── llama-server
│
├── models/
│   └── smollm2-1.7b-instruct-q4_k_m.gguf
│
├── smollm-template.jinja
│
└── start-ai.sh
```
---

# Segurança

Por padrão, utilizamos:

--host {seu_ip}

Isso mantém o servidor acessível somente pelo próprio dispositivo.

A interface pode ser acessada através de:

http://{seu_ipA}:8080

«_Não exponha o "llama-server" diretamente à internet._»

Se posteriormente você quiser acessar a IA a partir de outro computador ou celular da mesma rede, será necessário configurar o servidor para aceitar conexões externas e considerar medidas adicionais de segurança.

---

# Limitações

Este projeto possui algumas limitações.

## Hardware

Smartphones não possuem o mesmo desempenho de computadores modernos.

## Modelo

O SmolLM2 1.7B é relativamente pequeno. Ele pode ser útil para tarefas simples, mas possui limitações quando comparado a modelos maiores.

## Internet

A IA executada localmente não possui conhecimento atualizado da internet automaticamente.

## Temperatura

Inferência contínua pode aumentar o consumo e a temperatura do aparelho.

## Armazenamento

Modelos maiores podem ocupar vários gigabytes.

---

## Trocar o modelo

O "llama.cpp" suporta diversos modelos no formato GGUF.

Para utilizar outro modelo, substitua:
```
-m ~/models/smollm2-1.7b-instruct-q4_k_m.gguf
```
pelo caminho do novo arquivo.

Por exemplo:
```
-m ~/models/outro-modelo.gguf
```
«_Importante: nem todo modelo possui o mesmo Chat Template. Ao trocar de modelo, verifique o template recomendado pelo autor._»

---

# Solução de problemas

## "aarch64" não aparece

Execute:
```
uname -m
```
Este tutorial foi desenvolvido principalmente para dispositivos ARM64.

---

##  "llama-server" não existe

Verifique:
```
ls ~/llama.cpp/build/bin/
```
Se necessário, recompile:
```
cd ~/llama.cpp
cmake -B build
cmake --build build --config Release -j$(nproc)
```
---

## "Permission denied"

Execute:
```
chmod +x ~/start-ai.sh
```
---

## "--system-prompt" não funciona

Algumas versões do "llama.cpp" não possuem essa opção.

Neste projeto utilizamos:
```
--chat-template-file ~/smollm-template.jinja
```
para carregar o template personalizado.

Você pode verificar as opções disponíveis na sua versão:
```
llama-server --help
```
---

## O modelo está muito lento

Verifique:
```
nproc
```
e:
```
free -h
```
Depois experimente diferentes valores de:
```
-t
-b
-c
```
Não existe uma configuração universal para todos os celulares.

---

# Tecnologias utilizadas

Tecnologia| Função
|----|-----|
"Termux" (https://termux.dev/)| Ambiente Linux no Android
"llama.cpp" (https://github.com/ggml-org/llama.cpp)| Inferência de LLM
"Hugging Face" (https://huggingface.co/)| Hospedagem de modelos
SmolLM2| Modelo de linguagem
GGUF| Formato do modelo
Jinja| Chat Template
Fish| Shell opcional

---

# Objetivo

O objetivo deste projeto é demonstrar que é possível executar uma IA local em um smartphone Android, utilizando ferramentas abertas, modelos quantizados e uma configuração relativamente leve.

A ideia não é transformar um celular em um servidor de IA de alto desempenho, mas demonstrar até onde um dispositivo móvel pode chegar utilizando modelos pequenos e otimizações.

---

# Créditos

Este projeto utiliza:

- "Termux" (https://termux.dev/)
- "llama.cpp" (https://github.com/ggml-org/llama.cpp)
- "Hugging Face" (https://huggingface.co/)
- SmolLM2

---

## Licença

> By Darlyson and [ChatGPT](chatgpt.com)

> Consulte as licenças individuais de cada software e modelo utilizado neste projeto.

> O código de configuração e os scripts deste repositório podem ser distribuídos conforme a licença definida pelo autor do projeto.
