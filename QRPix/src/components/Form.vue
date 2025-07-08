<script setup lang="ts">
import { ref } from 'vue';

const chavePix = ref('');
const nome = ref('');
const cidade = ref('');
const valor = ref('');
const descricao = ref('');
const qrCodeUrl = ref('');
const pixCodeDebug = ref('');

const mostrarPopUp = ref(false);
const mensagemPopup = ref('');
const tipoPopup = ref('erro');

function mostrarMensagem(mensagem: string, tipo: string = 'erro') {
    mostrarPopUp.value = true;
    mensagemPopup.value = mensagem;
    tipoPopup.value = tipo;
    
    setTimeout(() => {
        mostrarPopUp.value = false;
    }, 2000);
}

function fecharPopUp() {
    mostrarPopUp.value = false;
}

function limparCaracteres(texto: string): string {
    return texto
        .normalize('NFD')
        .replace(/[\u0300-\u036f]/g, '')
        .replace(/[^A-Za-z0-9\s]/g, '')
        .trim()
        .toUpperCase();
}

function calcularCRC16(payload: string): string {
    const data = payload + '6304';
    const polinomio = 0x1021;
    let crc = 0xFFFF;

    for (let i = 0; i < data.length; i++) {
        crc ^= (data.charCodeAt(i) << 8);
        for (let j = 0; j < 8; j++) {
            if (crc & 0x8000) {
                crc = (crc << 1) ^ polinomio;
            } else {
                crc = crc << 1;
            }
            crc &= 0xFFFF;
        }
    }
    
    return crc.toString(16).toUpperCase().padStart(4, '0');
}

function gerarPixEMV(): string {
    try {
        console.log('Iniciando geração PIX EMV')

        let pixString = ''

        pixString += '000201'

        pixString += '010212'

        const pixKeyLength = chavePix.value.length.toString().padStart(2, '0')
        const pixKeyField = `01${pixKeyLength}${chavePix.value}`
        const merchantAccountInfo = `0014BR.GOV.BCB.PIX${pixKeyField}`
        pixString += `26${merchantAccountInfo.length.toString().padStart(2, '0')}${merchantAccountInfo}`

        pixString += '52040000'

        pixString += '5303986'

        if (valor.value && parseFloat(valor.value) > 0) {
            const valorFormatado = parseFloat(valor.value).toFixed(2)
            pixString += `54${valorFormatado.length.toString().padStart(2, '0')}${valorFormatado}`
        }

        pixString += '5802BR'

        const nomeFormatado = limparCaracteres(nome.value).substring(0, 25)
        pixString += `59${nomeFormatado.length.toString().padStart(2, '0')}${nomeFormatado}`

        const cidadeFormatada = limparCaracteres(cidade.value).substring(0, 15)
        pixString += `60${cidadeFormatada.length.toString().padStart(2, '0')}${cidadeFormatada}`

        if (descricao.value && descricao.value.trim()) {
            const descricaoFormatada = descricao.value.substring(0, 25)
            const additionalDataField = `05${descricaoFormatada.length.toString().padStart(2, '0')}${descricaoFormatada}`
            pixString += `62${additionalDataField.length.toString().padStart(2, '0')}${additionalDataField}`
        }

        const crc = calcularCRC16(pixString)
        pixString += `6304${crc}`

        console.log('PIX final gerado:', pixString)
        console.log('Tamanho:', pixString.length)
        
        if (pixString.length < 50 || pixString.length > 500) {
            throw new Error('PIX gerado tem tamanho inválido')
        }

        return pixString
    } catch (error) {
        console.error('Erro ao gerar PIX:', error)
        throw error
    }
}

function gerarQRCode() {
    try {
        const pixChaveLimpa = chavePix.value?.trim();
        const nomeLimpo = nome.value?.trim();
        const cidadeLimpa = cidade.value?.trim();

        if (!pixChaveLimpa || !nomeLimpo || !cidadeLimpa) {
            mostrarMensagem('Por favor, preencha todos os campos obrigatórios.', 'erro');
            return;
        }

        // Validações adicionais
        if (pixChaveLimpa.length < 1 || pixChaveLimpa.length > 77) {
            mostrarMensagem('Chave PIX deve ter entre 1 e 77 caracteres.', 'erro');
            return;
        }

        if (nomeLimpo.length < 1 || nomeLimpo.length > 25) {
            mostrarMensagem('Nome deve ter entre 1 e 25 caracteres.', 'erro');
            return;
        }

        if (cidadeLimpa.length < 1 || cidadeLimpa.length > 15) {
            mostrarMensagem('Cidade deve ter entre 1 e 15 caracteres.', 'erro');
            return;
        }

        if (valor.value && (parseFloat(valor.value) < 0 || parseFloat(valor.value) > 99999999.99)) {
            mostrarMensagem('Valor deve estar entre 0 e 99.999.999,99.', 'erro');
            return;
        }

        const pixEMV = gerarPixEMV()
        pixCodeDebug.value = pixEMV;
        
        const tamanho = '300x300'
        const urlApi = `https://api.qrserver.com/v1/create-qr-code/?size=${tamanho}&data=${encodeURIComponent(pixEMV)}`

        qrCodeUrl.value = urlApi;

        mostrarMensagem('QR Code gerado com sucesso!', 'sucesso');

    } catch (error) {
        console.error('Erro ao gerar QR Code:', error)
        mostrarMensagem('Erro ao gerar QR Code. Verifique os campos e tente novamente.', 'erro');
        return;
    }
}

function limparTudo() {
    chavePix.value = ''
    nome.value = ''
    cidade.value = ''
    valor.value = ''
    descricao.value = ''
    qrCodeUrl.value = ''
    mostrarMensagem('Formulário limpo!', 'sucesso')
}
</script>

<template>
    <div class="container">
        <div v-if="mostrarPopUp" class="popup-overlay" @click="fecharPopUp">
            <div class="popup" :class="tipoPopup" @click.stop>
                <div class="popup-header">
                    <span class="popup-icon">
                        {{ tipoPopup === 'erro' ? '❌' : '✅' }}
                    </span>
                    <button @click="fecharPopUp" class="popup-close">×</button>
                </div>
                <div class="popup-content">
                    <p>{{ mensagemPopup }}</p>
                </div>
            </div>
        </div>
        <div class="header">
            <h1>Gerador de QR Code PIX</h1>
            <p>Gere um QR Code PIX com os dados fornecidos</p>
        </div>


        <div class="formulario">
            <h2>📝 Preencha os dados:</h2>

            <div class="campo">
                <label>Chave PIX *:</label>
                <input v-model="chavePix" type="text" placeholder="CPF, CNPJ, email, telefone ou chave aleatória" />
            </div>

            <div class="campo">
                <label>Seu Nome *:</label>
                <input v-model="nome" type="text" placeholder="Digite seu nome completo" />
            </div>

            <div class="campo">
                <label>Cidade *:</label>
                <input v-model="cidade" type="text" placeholder="Digite sua cidade" />
            </div>

            <div class="campo">
                <label>Valor (opcional):</label>
                <input v-model="valor" type="number" placeholder="0.00" step="0.01" min="0" />
                <small>Deixe em branco para valor livre</small>
            </div>

            <div class="campo">
                <label>Descrição (opcional):</label>
                <input v-model="descricao" type="text" placeholder="Descrição do pagamento" />
            </div>
            
        </div>
        <div class="botoes">
        <button @click="gerarQRCode" class="botao-principal">
          ✨ Gerar QR Code PIX
        </button>
        <button @click="limparTudo" class="botao-limpar">
        Limpar
        </button>
      </div>

      <div v-if="qrCodeUrl" class="resultado">
        <h2>Seu código de pagamento foi gerado com sucesso!</h2>
        <div class="qr-box">
            <img :src="qrCodeUrl" alt="QR Code PIX">
        </div>
        <div class="info">
            <p><strong>Chave PIX:</strong> {{ chavePix }}</p>
            <p><strong>Nome:</strong> {{ nome }}</p>
            <p><strong>Cidade:</strong> {{ cidade }}</p>
            <p v-if="valor && parseFloat(valor) > 0"><strong>Valor:</strong> R$ {{ parseFloat(valor).toFixed(2) }}</p>
            <p v-else><strong>Valor:</strong> Livre (a definir no pagamento)</p>
            <p v-if="descricao"><strong>Descrição:</strong> {{ descricao }}</p>
        </div>
        <div class="debug-info">
            <details>
                <summary>🔍 Ver código PIX (debug)</summary>
                <p style="word-break: break-all; font-family: monospace; font-size: 12px; background: #f0f0f0; color: #000; padding: 10px; border-radius: 5px;">
                    {{ pixCodeDebug }}
                </p>
            </details>
        </div>
      </div>
    </div>
</template>

<style scoped>
</style>