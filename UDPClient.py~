#-*-coding:utf-8-*-
import sys
import socket
import select
import time
import math
import hashlib
from optparse import OptionParser
from threading import Thread
from threading import Condition
from random import randint

"Classe Transmissor"
class Transmissor:
	def __init__(self, porta, endereco, taxa, tipo):
		"Atributos iniciais"
		self.sock = self.iniciaCliente()
		self.porta = porta
		self.endereco = endereco
		self.taxa = taxa
		self.tipo = tipo
		self.enviando = True
		
		if tipo == SNW:
			self.tamanhoJanela = 1
		elif tipo == GBN:
			self.tamanhoJanela = TAMANHO_JANELA
		else:
			self.tamanhoJanela = TAMANHO_JANELA
		self.filaCircular = self.BufferCircular(self.tamanhoJanela) 
		self.condition = Condition()
		self.ultimoQuadroEnviado = 0
		self.ultimoACKRecebido = 0
		self.numQuadrosReenviados = 0
	
	"Classe Buffer Circular - Simula Janela Deslizante"
	class BufferCircular:
		def __init__(self, tamanho):
			self.data = [None for i in xrange(tamanho)]
			self.tamanho = tamanho

		"Insere na última posição e retira a primeira"
		def insere(self, x):
			self.data.pop(0)
			self.data.append(x)
		
		"Define primeira posição estar livre"
		def liberaEspaco(self):
			self.data.pop(0)
			self.data.insert(0, None)
		
		"Define posição 'posicao' estar livre"
		def liberaEspacoPosicao(self, posicao):
			self.data.pop(posicao)
			self.data.insert(posicao, None)
		
		"Verifica se tem espaço livre no buffer"
		def temEspacoLivre(self):
			return self.data[0] == None
		
		"Verifica se buffer está vazio"
		def bufferEstaVazio(self):
			return self.data[0] == ""
		
		"Retorna o primeiro elemento do buffer"
		def primeiro(self):
			return self.data[0]
		
		"Retorna o último elemento do buffer"
		def ultimo(self):
			return self.data[self.tamanho - 1]
		
		"Retorna o buffer"
		def get(self):
			return self.data
	
	"Retorna o buffer do transmissor"	
	def getBuffer(self):
		return self.filaCircular
	
	"Retorna a condição utilizada nas threads"
	def getCondition(self):
		return self.condition
	
	"Retorna o tamanho da janela"
	def getTamanhoJanela(self):
		return self.tamanhoJanela
	
	"Retorna o socket"
	def getSock(self):
		return self.sock
	
	"Retorna o último quadro enviado"
	def getLFS(self):
		return self.ultimoQuadroEnviado
	
	"Retorna o último quadro recebido"
	def getLAR(self):
		return self.ultimoACKRecebido
	
	"Retorna a taxa de perda de pacotes"
	def getTaxa(self):
		return self.taxa
	
	"Retorna o tipo de transmissão"
	def getTipo(self):
		return self.tipo
	
	"Retorna o número de quadros enviados"
	def getNumQuadrosReenviados(self):
		return self.numQuadrosReenviados
	
	"Incrementa o número de quadros reenviados"
	def setNumQuadrosReenviados(self):
		self.numQuadrosReenviados += 1
	
	"Seta o último quadro enviado"
	def setLFS(self, LFS):
		self.ultimoQuadroEnviado = LFS
	
	"Seta o último quadro recebido"
	def setLAR(self, LAR):
		self.ultimoACKRecebido = LAR
	
	"Inicia a conexão do cliente"
	def iniciaCliente(self):
		try:
			sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
			sock.setblocking(0)
		except socket.error:
			print "Falha ao criar socket"
			sys.exit()
	
		return sock
	
	"Envia uma mensagem ao servidor"
	def enviaMensagem(self, mensagem):
		self.sock.sendto(mensagem, (self.endereco, self.porta))
	
	"Coloca o cabeçalho na mensagem"
	def colocaCabecalho(self, mensagem, numQuadro):
		quadro = "{0:0>8}".format(numQuadro)
		checksumMD5 = str(self.geraMD5Mensagem(quadro + mensagem))		
		mensagem = checksumMD5 + quadro + mensagem
		
		return mensagem
	
	"Gera MD5 para arquivo de entrada"
	def geraMD5(self, arquivo):
		return hashlib.md5(open(arquivo).read()).hexdigest()
	
	"Gera MD5 para a mensagem enviada"
	def geraMD5Mensagem(self, mensagem):
		return hashlib.md5(mensagem).hexdigest()
	
	"Confere se o MD5 da mensagem é o mesmo informado no cabeçalho"
	def confereMD5Mensagem(self, mensagem, MD5):
		return str(hashlib.md5(mensagem).hexdigest()) == MD5
	
	"Seta se o cliente está enviando ou não"
	def setEnviando(self, envioAtivo):
		self.enviando = envioAtivo
	
	"Retorna se o cliente está enviando"
	def envioAtivo(self):
		return self.enviando

"Classe para fazer leitura do arquivo"
class LeituraArquivo(Thread):

	def __init__(self, entrada, transmissor):
		"Atributos iniciais"
		Thread.__init__(self)
		self.entrada = entrada
		self.condition = transmissor.getCondition()
		self.filaCircular = transmissor.getBuffer()
		self.transmissor = transmissor
		
	def run(self):
		msgLida = False
		file = open(self.entrada, 'r')
		while True:
			self.condition.acquire()
			while True:
				"Caso não tenha lido a mensagem anteriormente, a lê"
				if not msgLida:
					msg = file.read(TAMANHO_DADOS)
		        
				"Verifica se tem espaço no buffer"
				temEspaco = self.filaCircular.temEspacoLivre()
				if temEspaco:
					"Caso sim, insere a mensagem"
					self.filaCircular.insere(msg)
					msgLida = False
				else :
					"Caso não, para a leitura"
					msgLida = True
					break
			"Caso o buffer esteja vazio, para a transmissão"
			if self.filaCircular.bufferEstaVazio():
				break
			"Caso contrário avisa thread de Envio e libera a condição"
			self.condition.notify()
			self.condition.release()
		
		"Ao fim da transmissão, avisa ter parado o envio, libera condição"
		self.transmissor.setEnviando(False)
		self.condition.notify()
		self.condition.release()
		print "Cliente finalizando ..."
		print "Quadros reenviados:", self.transmissor.getNumQuadrosReenviados()
		file.close
        
class Encapsulamento(Thread):

	def __init__(self, transmissor, entrada):
		"Atributos iniciais"
		Thread.__init__(self)
		self.condition = transmissor.getCondition()
		self.filaCircular = transmissor.getBuffer()
		self.sock = transmissor.getSock()
		self.transmissor = transmissor
		self.goBackN = transmissor.getTipo() == GBN
		self.entrada = entrada
		self.taxa, self.limite = transmissor.getTaxa()

	def run(self):
		numQuadro = 0
		socket = [self.sock]
		primeiroEnvio = True
		NACK = True
		enviarMensagem = False
		ACKsRecebidos = self.transmissor.BufferCircular(2 * self.transmissor.getTamanhoJanela())
		mensagensEnviadas = self.transmissor.BufferCircular(self.transmissor.getTamanhoJanela())
		
		while True:
			self.condition.acquire()
			while True:
				temEspaco = self.filaCircular.temEspacoLivre()
				breakLoop = False
				"Caso o buffer não tenha espaço livre"
				if temEspaco == False:
					"Caso seja o primeiro envio"
					if primeiroEnvio:
						"Caso não tenha recebido ACK da mensagem inicial"
						if NACK:
							"Caso a transmissão seja Go Back N"
							if self.goBackN:
								"Escreve mensagem inicial com transmissão GO Back N e checksum do arquivo"
								mensagemInicial = str(1) + str(self.transmissor.geraMD5(self.entrada))
								mensagemInicial = self.transmissor.colocaCabecalho(mensagemInicial, numQuadro)
							else:
								"Escreve mensagem inicial com transmissão não sendo GO Back N e checksum do arquivo"
								mensagemInicial = str(0) + str(self.transmissor.geraMD5(self.entrada))
								mensagemInicial = self.transmissor.colocaCabecalho(mensagemInicial, numQuadro)
							
							"Envia mensagem inicial"
							self.transmissor.enviaMensagem(mensagemInicial)
							ent, sai, exc = select.select(socket, [], [], TIMEOUT)
							"Verifica se recebeu resposta"
							if ent:
								resposta, endereco = self.sock.recvfrom(TAMANHO_CABECALHO)
								if self.receberMensagem():
									MD5Mensagem = resposta[:32]
									quadro = int(resposta[32:40])
									if not self.transmissor.confereMD5Mensagem(resposta[32:40], MD5Mensagem):
										print "MD5 não confere"
										continue

									print "Conexão iniciada"
									tempoInicio = time.time()
									NACK = False
								else:
									print "Mensagem inicial não recebida"
									continue
#							"Caso não tenha recebido resposta, temporização dispara"
							else:
								print "Expirou timeout"
								continue
						
						"Recebe dados lidos do buffer"		
						dados = self.filaCircular.get()
						
						"Envia a primeira janela de dados"
						for i in xrange(self.transmissor.getTamanhoJanela()):
							numQuadro += 1
							print "Enviando", numQuadro
							mensagem = self.transmissor.colocaCabecalho(dados[i], numQuadro)
							if not dados[i] == '':
								self.transmissor.enviaMensagem(mensagem)
							mensagensEnviadas.insere(mensagem)
							self.transmissor.setLFS(numQuadro)
						primeiroEnvio = False

#					"Caso não seja o primeiro envio"
					else:
						"Envia todos os dados possíveis dentro da janela"
						while (self.transmissor.getLFS() - self.transmissor.getLAR()) < self.transmissor.getTamanhoJanela() and enviarMensagem:
							numQuadro += 1
							print "Enviando", numQuadro
							mensagem = self.transmissor.colocaCabecalho(dados[self.transmissor.getLFS() - self.transmissor.getLAR()], numQuadro)
							if not dados[self.transmissor.getLFS() - self.transmissor.getLAR()] == '':
								self.transmissor.enviaMensagem(mensagem)
							mensagensEnviadas.insere(mensagem)
							self.transmissor.setLFS(numQuadro)
						
						"Verifica recebimento da resposta"
						entrada, saida, excecao = select.select(socket, [], [], TIMEOUT)
						if entrada:
							resposta, endereco = self.sock.recvfrom(TAMANHO_CABECALHO)
							quadro = int(resposta[32:40]) #Não importa
							if self.receberMensagem():
								MD5Mensagem = resposta[:32]
								quadro = int(resposta[32:40]) #Não importa
								
								if not self.transmissor.confereMD5Mensagem(resposta[32:40], MD5Mensagem):
									print "MD5 não confere"
									continue
								
#								"Caso o tipo não seja Go Back N, apenas insere quadro em ACKs recebidos"
								if not self.goBackN:
									ACKsRecebidos.insere(quadro)
#								"Caso contrário, verifica se já não recebeu o quadro antes"
								else:
#									if quadro == self.transmissor.getLAR() + 1:
									if not quadro in ACKsRecebidos.get():
										ACKsRecebidos.insere(quadro)
									else:
										print "Quadro já recebido", quadro
										continue
								
#								"Atualiza o último quadro recebido e libera a devida posição no buffer"
								if self.transmissor.getLAR() + 1 in ACKsRecebidos.get():
									while self.transmissor.getLAR() + 1 in ACKsRecebidos.get():
										self.transmissor.setLAR(self.transmissor.getLAR() + 1)
									self.filaCircular.liberaEspaco()
									enviarMensagem = True
									breakLoop = True
								else:
									self.filaCircular.liberaEspacoPosicao(quadro - (self.transmissor.getLAR() + 1))
									enviarMensagem = False
							else:
								print "Mensagem não recebida", quadro							
#						"Caso mensagem não seja recebida temporização dispara"
						else:
							print "Expirou timeout"
#							"Para Go Back N reenvia todas as mensagens"
							if self.goBackN:
								mensagensRepetidas = mensagensEnviadas.get()
								for i in xrange(len(mensagensRepetidas)):
									if len(mensagensRepetidas[i]) > TAMANHO_CABECALHO:
										msg = mensagensRepetidas[i]
										quadroReenviado = int(msg[32:40])
										print "Reenviando", quadroReenviado
										self.transmissor.enviaMensagem(mensagensRepetidas[i])
									self.transmissor.setNumQuadrosReenviados()
#							"Para outros tipos, reenvia apenas a mensagem com erro"
							else:
								if len(mensagensEnviadas.primeiro()) > TAMANHO_CABECALHO:		
									msg = mensagensEnviadas.primeiro()
									quadroReenviado = int(msg[32:40])
									print "Reenviando", quadroReenviado
									self.transmissor.enviaMensagem(mensagensEnviadas.primeiro())
								self.transmissor.setNumQuadrosReenviados()
							enviarMensagem = False
#				"Verifica se é para sair do loop de envio"
				if breakLoop:
					break
			"Aguarda notificação da thread de envio e libera condição"
			self.condition.wait()
			self.condition.release()
			
			"Caso a transmissão tenha acabado, conta tempo total e finaliza thread"
			if not self.transmissor.envioAtivo():
				tempoFinal = time.time()
				tempoTotal = tempoFinal - tempoInicio
				print "Tempo de transmissão:", tempoTotal
				break
	
	"Sorteia se mensagem foi 'perdida'"
	def receberMensagem(self):
		random = randint(1, self.limite)
		return (random > self.taxa)

"Transforma número em ponto flutuante para inteiro para realizar sorteio"
def taxaInt(taxa):
	limite = 100
	taxaInt = math.floor(taxa)

	while not -0.01 <= taxa - taxaInt <= 0.01:
		taxa *= 10
		limite *= 10
		taxaInt = math.floor(taxa)
	
	return int(taxaInt), limite
	
"Faz o recebimento dos parâmetros por linha de comando"
def recebeParametros():	
	usage = "%prog -f Entrada -e Host -p Porta (-w Stop and wait OU -g Go Back N OU -a Selective ACK) -t % Perda de pacotes"
	parser = OptionParser(usage=usage, add_help_option=False)

	parser.add_option("-f", type="string", help="Nome do arquivo de entrada", dest="entrada")
	parser.add_option("-h", type="string", help="Nome ou endereco IP do servidor", dest="endereco")
	parser.add_option("-p", type="int", help="Numero da porta que o servidor esta aguardando conexoes", dest="porta")
	parser.add_option('-w', help='Define tipo de transmissao ser stop and wait', default=False, action='store_true', dest='snw')
	parser.add_option('-g', help='Define tipo de transmissao ser go back n', default=False, action='store_true', dest='gbn')
	parser.add_option('-a', help='Define tipo de transmissao ser selective ACK', default=False, action='store_true', dest='sack')
	parser.add_option("-t", type="float", help="Especifica a taxa de perda de pacotes em %", dest="taxa")

	(options, args) = parser.parse_args()
	
	arqEntrada = options.entrada
	endereco = options.endereco
	porta = options.porta
	stopNWait = options.snw
	goBackN = options.gbn
	selectiveACK = options.sack
	taxa = options.taxa

	# Se faltar algum argumento, nenhum tipo de transmissao for escolhido ou mais de um for escolhido, termina com erro
	if ((arqEntrada == None or endereco == None or porta == None or taxa == None)
		or (stopNWait == goBackN == selectiveACK == False)
		or ((stopNWait == goBackN == True) or (stopNWait == selectiveACK == True) or (selectiveACK == goBackN == True))):
		print usage
		print "Erro ao ler argumentos. Todos os argumentos para o programa foram passados?"
		sys.exit()
	
	if stopNWait:
		tipo = 1
	elif goBackN:
		tipo = 2
	else:
		tipo = 3
	
	print ("Entrada : %s " % (options.entrada))
	print ("Host : %s " % (options.endereco))
	print ("Porta : %s " % (options.porta))
	print ("Stop and Wait : %s " % (options.snw))
	print ("Go back N : %s " % (options.gbn))
	print ("Selective ACK : %s " % (options.sack))
	print ("Tipo : %s" % (tipo))
	print ("Taxa : %s " % (options.taxa))

	return arqEntrada, endereco, porta, tipo, taxa
	
"Método principal"
def main():
	arqEntrada, endereco, porta, tipo, taxa = recebeParametros()
	
	limitesTaxa = taxaInt(taxa)
	
	transmissor = Transmissor(porta, endereco, limitesTaxa, tipo)
    
	threadLeitura = LeituraArquivo(arqEntrada, transmissor)
	threadEncapsulamento = Encapsulamento(transmissor, arqEntrada)
    
	threadLeitura.start()
	threadEncapsulamento.start()
	
	threadLeitura.join()
	threadEncapsulamento.join()

"Variáveis globais e chamada do método principal"
if __name__ == '__main__':
	SNW = 1
	GBN = 2
	SACK = 3
	TAMANHO_PACOTE = 1460
	TAMANHO_DADOS = 1420
	TAMANHO_CABECALHO = 40
	TAMANHO_JANELA = 10
	TIMEOUT = 2
	main()
