<script setup lang="ts">
import { ref } from 'vue';

const chavePix = ref('');
const nome = ref('');
const cidade = ref('');
const valor = ref('');
const descricao = ref('');
const qrCodeUrl = ref('');

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

function calcularCRC16(payload: string): string {
    const data = payload + '6304';
    const polinomio = 0x1021
    let resultado = 0xFFFF

    for (let i = 0; i < data.length; i++) {
        resultado ^= (data.charCodeAt(i) << 8)
        for (let j = 0; j < 8; j++) {
            if (resultado & 0x8000) {
                resultado = (resultado << 1) ^ polinomio
            } else {
                resultado = resultado << 1
            }
            resultado &= 0xFFFF
        }
    }
    return resultado.toString(16).toUpperCase().padStart(4, '0')
}

// Alguns console.log para debug
function gerarPixEMV(): string {
    try {
        console.log('Iniciando geração PIX EMV')

        let pixString = ''

        pixString += '000201'
        console.log('Após 00:', pixString)

        const pixKeyLength = chavePix.value.length.toString().padStart(2, '0')
        const pixKeyData = `01${pixKeyLength}${chavePix.value}`
        const merchantAccountInfo = `0014BR.GOV.BCB.PIX${pixKeyData}`
        pixString += `26${merchantAccountInfo.length.toString().padStart(2, '0')}${merchantAccountInfo}`
        console.log('Após 26:', pixString)

        pixString += '52040000'
        console.log('Após 52:', pixString)

        pixString += '5303986'
        console.log('Após 53:', pixString)

        if (valor.value && parseFloat(valor.value) > 0) {
            const valorFormatado = parseFloat(valor.value).toFixed(2)
            pixString += `54${valorFormatado.length.toString().padStart(2, '0')}${valorFormatado}`
            console.log('Após 54:', pixString)
        }

        pixString += '5802BR'
        console.log('Após 58:', pixString)

        const nomeFormatado = nome.value.toUpperCase()
        pixString += `59${nomeFormatado.length.toString().padStart(2, '0')}${nomeFormatado}`
        console.log('Após 59:', pixString)

        const cidadeFormatada = cidade.value.toUpperCase()
        pixString += `60${cidadeFormatada.length.toString().padStart(2, '0')}${cidadeFormatada}`
        console.log('Após 60:', pixString)

        if (descricao.value) {
            const descricaoFormatada = descricao.value.toUpperCase()
            const additionalDataField = `05${descricaoFormatada.length.toString().padStart(2, '0')}${descricaoFormatada}`
            pixString += `62${additionalDataField.length.toString().padStart(2, '0')}${additionalDataField}`
            console.log('Após 62:', pixString)
        }

        console.log('Calculando CRC16 para:', pixString)
        const crc = calcularCRC16(pixString)
        console.log('CRC16 calculado:', crc)
        pixString += `6304${crc}`

        console.log('PIX final gerado:', pixString)
        console.log('PIX EMV gerado com sucesso')

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
        const pixEMV = gerarPixEMV()
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
          🗑️ Limpar
        </button>
      </div>
    </div>
</template>

<style scoped></style>