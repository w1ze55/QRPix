<script setup lang="ts">
import { ref, onMounted } from 'vue';

// Configurações do ambiente
const config = {
    qrApi: {
        url: import.meta.env.VITE_QR_API_URL || 'https://api.qrserver.com/v1/create-qr-code/',
        size: import.meta.env.VITE_QR_SIZE || '300x300'
    },
    pix: {
        countryCode: import.meta.env.VITE_PIX_COUNTRY_CODE || 'BR',
        currencyCode: import.meta.env.VITE_PIX_CURRENCY_CODE || '986',
        merchantCategory: import.meta.env.VITE_PIX_MERCHANT_CATEGORY || '0000'
    },
    limits: {
        maxNameLength: Number(import.meta.env.VITE_PIX_MAX_NAME_LENGTH) || 25,
        maxCityLength: Number(import.meta.env.VITE_PIX_MAX_CITY_LENGTH) || 15,
        maxDescriptionLength: Number(import.meta.env.VITE_PIX_MAX_DESCRIPTION_LENGTH) || 25
    },
    app: {
        title: import.meta.env.VITE_APP_TITLE || 'Gerador de QR Code PIX',
        description: import.meta.env.VITE_APP_DESCRIPTION || 'Gere códigos PIX instantâneos de forma rápida e segura'
    },
    defaults: {
        pixKey: import.meta.env.VITE_DEFAULT_PIX_KEY || '',
        name: import.meta.env.VITE_DEFAULT_NAME || '',
        city: import.meta.env.VITE_DEFAULT_CITY || ''
    },
    debug: import.meta.env.VITE_DEBUG_MODE === 'true'
};

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

// Inicializar campos com valores padrão do .env
onMounted(() => {
    chavePix.value = config.defaults.pixKey;
    nome.value = config.defaults.name;
    cidade.value = config.defaults.city;
    
    if (config.debug) {
        console.log('Configurações carregadas:', config);
    }
});

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
        if (config.debug) {
            console.log('Iniciando geração PIX EMV com configurações:', config.pix);
        }

        let pixString = ''

        pixString += '000201'
        pixString += '010212'

        const pixKeyLength = chavePix.value.length.toString().padStart(2, '0')
        const pixKeyField = `01${pixKeyLength}${chavePix.value}`
        const merchantAccountInfo = `0014BR.GOV.BCB.PIX${pixKeyField}`
        pixString += `26${merchantAccountInfo.length.toString().padStart(2, '0')}${merchantAccountInfo}`

        pixString += `5204${config.pix.merchantCategory}`

        pixString += `53${config.pix.currencyCode.length.toString().padStart(2, '0')}${config.pix.currencyCode}`

        if (valor.value && parseFloat(valor.value) > 0) {
            const valorFormatado = parseFloat(valor.value).toFixed(2)
            pixString += `54${valorFormatado.length.toString().padStart(2, '0')}${valorFormatado}`
        }

        pixString += `58${config.pix.countryCode.length.toString().padStart(2, '0')}${config.pix.countryCode}`

        const nomeFormatado = limparCaracteres(nome.value).substring(0, config.limits.maxNameLength)
        pixString += `59${nomeFormatado.length.toString().padStart(2, '0')}${nomeFormatado}`

        const cidadeFormatada = limparCaracteres(cidade.value).substring(0, config.limits.maxCityLength)
        pixString += `60${cidadeFormatada.length.toString().padStart(2, '0')}${cidadeFormatada}`

        if (descricao.value && descricao.value.trim()) {
            const descricaoFormatada = descricao.value.substring(0, config.limits.maxDescriptionLength)
            const additionalDataField = `05${descricaoFormatada.length.toString().padStart(2, '0')}${descricaoFormatada}`
            pixString += `62${additionalDataField.length.toString().padStart(2, '0')}${additionalDataField}`
        }

        const crc = calcularCRC16(pixString)
        pixString += `6304${crc}`

        if (config.debug) {
            console.log('PIX final gerado:', pixString);
            console.log('Tamanho:', pixString.length);
        }
        
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

        // Validações usando configurações do .env
        if (pixChaveLimpa.length < 1 || pixChaveLimpa.length > 77) {
            mostrarMensagem('Chave PIX deve ter entre 1 e 77 caracteres.', 'erro');
            return;
        }

        if (nomeLimpo.length < 1 || nomeLimpo.length > config.limits.maxNameLength) {
            mostrarMensagem(`Nome deve ter entre 1 e ${config.limits.maxNameLength} caracteres.`, 'erro');
            return;
        }

        if (cidadeLimpa.length < 1 || cidadeLimpa.length > config.limits.maxCityLength) {
            mostrarMensagem(`Cidade deve ter entre 1 e ${config.limits.maxCityLength} caracteres.`, 'erro');
            return;
        }

        if (valor.value && (parseFloat(valor.value) < 0 || parseFloat(valor.value) > 99999999.99)) {
            mostrarMensagem('Valor deve estar entre 0 e 99.999.999,99.', 'erro');
            return;
        }

        if (descricao.value && descricao.value.length > config.limits.maxDescriptionLength) {
            mostrarMensagem(`Descrição deve ter no máximo ${config.limits.maxDescriptionLength} caracteres.`, 'erro');
            return;
        }

        const pixEMV = gerarPixEMV()
        pixCodeDebug.value = pixEMV;
        
        const urlApi = `${config.qrApi.url}?size=${config.qrApi.size}&data=${encodeURIComponent(pixEMV)}`

        qrCodeUrl.value = urlApi;

        mostrarMensagem('QR Code gerado com sucesso!', 'sucesso');

    } catch (error) {
        console.error('Erro ao gerar QR Code:', error)
        mostrarMensagem('Erro ao gerar QR Code. Verifique os campos e tente novamente.', 'erro');
        return;
    }
}

function limparTudo() {
    chavePix.value = config.defaults.pixKey;
    nome.value = config.defaults.name;
    cidade.value = config.defaults.city;
    valor.value = '';
    descricao.value = '';
    qrCodeUrl.value = '';
    mostrarMensagem('Formulário limpo!', 'sucesso');
}
</script>

<template>
    <div class="app-container">
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

        <div class="main-container">
            <div class="header-section">
                <div class="logo-container">
                    <img src="/src/assets/Logo.png" alt="QRPix Logo" class="logo" />
        </div>
                <h1 class="main-title">{{ config.app.title }}</h1>
                <p class="subtitle">{{ config.app.description }}</p>
            </div>

            <div class="content-layout" :class="{ 'has-result': qrCodeUrl, 'centered': !qrCodeUrl }">
                <div class="form-container">
                    <div class="form-card">
                        <h2 class="form-title">
                            <span class="form-icon">📱</span>
                            Dados do PIX
                        </h2>

                        <div class="form-grid">
                            <div class="input-group">
                                <label class="input-label">
                                    <span class="label-text">Chave PIX</span>
                                    <span class="required">*</span>
                                </label>
                                <input 
                                    v-model="chavePix" 
                                    type="text" 
                                    class="input-field"
                                    placeholder="CPF, CNPJ, email, telefone ou chave aleatória"
                                    required
                                />
            </div>

                            <div class="input-group">
                                <label class="input-label">
                                    <span class="label-text">Nome Completo</span>
                                    <span class="required">*</span>
                                </label>
                                <input 
                                    v-model="nome" 
                                    type="text" 
                                    class="input-field"
                                    placeholder="Digite seu nome completo"
                                    required
                                />
            </div>

                            <div class="input-group">
                                <label class="input-label">
                                    <span class="label-text">Cidade</span>
                                    <span class="required">*</span>
                                </label>
                                <input 
                                    v-model="cidade" 
                                    type="text" 
                                    class="input-field"
                                    placeholder="Digite sua cidade"
                                    required
                                />
            </div>

                            <div class="input-group">
                                <label class="input-label">
                                    <span class="label-text">Valor</span>
                                    <span class="optional">(opcional)</span>
                                </label>
                                <div class="input-wrapper">
                                    <span class="currency-symbol">R$</span>
                                    <input 
                                        v-model="valor" 
                                        type="number" 
                                        class="input-field currency-input"
                                        placeholder="0,00"
                                        step="0.01" 
                                        min="0"
                                    />
                                </div>
                                <small class="input-hint">Deixe em branco para valor livre</small>
            </div>
            
                            <div class="input-group full-width">
                                <label class="input-label">
                                    <span class="label-text">Descrição</span>
                                    <span class="optional">(opcional)</span>
                                </label>
                                <input 
                                    v-model="descricao" 
                                    type="text" 
                                    class="input-field"
                                    placeholder="Descrição do pagamento"
                                />
                            </div>
        </div>
                        
                        <div class="button-group">
                            <button @click="gerarQRCode" class="btn btn-primary">
                                <span class="btn-icon">✨</span>
                                Gerar QR Code PIX
        </button>
                            <button @click="limparTudo" class="btn btn-secondary">
                                <span class="btn-icon">🗑️</span>
                                Limpar Formulário
        </button>
      </div>
                    </div>
                </div>

                <div v-if="qrCodeUrl" class="result-container">
                    <div class="result-card">
                        <div class="result-header">
                            <h2 class="result-title">
                                <span class="success-icon">🎉</span>
                                QR Code gerado com sucesso!
                            </h2>
                        </div>
                        
                        <div class="qr-section">
                            <div class="qr-container">
                                <img :src="qrCodeUrl" alt="QR Code PIX" class="qr-image" />
                            </div>
                        </div>
                        
                        <div class="info-section">
                            <h3 class="info-title">Informações do PIX</h3>
                            <div class="info-grid">
                                <div class="info-item">
                                    <span class="info-label">Chave PIX:</span>
                                    <span class="info-value">{{ chavePix }}</span>
                                </div>
                                <div class="info-item">
                                    <span class="info-label">Nome:</span>
                                    <span class="info-value">{{ nome }}</span>
                                </div>
                                <div class="info-item">
                                    <span class="info-label">Cidade:</span>
                                    <span class="info-value">{{ cidade }}</span>
                                </div>
                                <div class="info-item">
                                    <span class="info-label">Valor:</span>
                                    <span class="info-value">
                                        {{ valor && parseFloat(valor) > 0 ? `R$ ${parseFloat(valor).toFixed(2)}` : 'Valor livre' }}
                                    </span>
                                </div>
                                <div v-if="descricao" class="info-item">
                                    <span class="info-label">Descrição:</span>
                                    <span class="info-value">{{ descricao }}</span>
                                </div>
                            </div>
        </div>
                        
                        <div class="debug-section">
                            <details class="debug-details">
                                <summary class="debug-summary">
                                    <span class="debug-icon">🔍</span>
                                    Ver código PIX (desenvolvedor)
                                </summary>
                                <div class="debug-content">
                                    <code class="debug-code">{{ pixCodeDebug }}</code>
        </div>
            </details>
                        </div>
                    </div>
                </div>
        </div>
      </div>
    </div>
</template>

<style>
/* Estilos globais para garantir background de tela cheia */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

html, body {
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
    overflow-x: hidden;
}

#app {
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
}
</style>

<style scoped>
* {
    box-sizing: border-box;
    margin: 0;
    padding: 0;
}

html, body {
    width: 100%;
    height: 100%;
    margin: 0;
    padding: 0;
    overflow-x: hidden;
}

.app-container {
    position: fixed;
    top: 0;
    left: 0;
    width: 100vw;
    height: 100vh;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    padding: 20px;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    display: flex;
    flex-direction: column;
    overflow: hidden;
}

.main-container {
    flex: 1;
    display: flex;
    flex-direction: column;
    width: 100%;
    max-width: 100%;
    overflow: hidden;
}

/* Header Section */
.header-section {
    text-align: center;
    margin-bottom: 25px;
    flex-shrink: 0;
}

.logo-container {
    margin-bottom: 15px;
}

.logo {
    width: 60px;
    height: 60px;
    object-fit: contain;
    filter: drop-shadow(0 4px 8px rgba(0,0,0,0.1));
}

.main-title {
    font-size: 2.2rem;
    font-weight: 700;
    color: white;
    margin: 0 0 8px 0;
    text-shadow: 0 2px 4px rgba(0,0,0,0.3);
}

.subtitle {
    font-size: 1.1rem;
    color: rgba(255,255,255,0.9);
    margin: 0;
    font-weight: 300;
}

/* Content Layout */
.content-layout {
    flex: 1;
    display: flex;
    gap: 30px;
    align-items: flex-start;
    justify-content: center;
    width: 100%;
    overflow-y: auto;
    overflow-x: hidden;
    min-height: 0;
}

.content-layout.has-result {
    justify-content: space-between;
    align-items: stretch;
}

.content-layout.centered {
    justify-content: stretch;
    align-items: center;
}

/* Form Container */
.form-container {
    flex: 1;
    display: flex;
    flex-direction: column;
    justify-content: center;
    min-width: 0;
    width: 100%;
    overflow: hidden;
}

.content-layout.has-result .form-container {
    flex: 1;
    justify-content: flex-start;
    max-width: 50%;
}

.content-layout.centered .form-container {
    max-width: none;
    width: 100%;
}

.form-card {
    background: white;
    border-radius: 20px;
    padding: 30px;
    box-shadow: 0 20px 40px rgba(0,0,0,0.1);
    backdrop-filter: blur(10px);
    width: 100%;
    overflow: hidden;
}

.content-layout.centered .form-card {
    padding: 40px;
    max-width: 1200px;
    margin: 0 auto;
}

.form-title {
    font-size: 1.6rem;
    font-weight: 600;
    color: #333;
    margin: 0 0 25px 0;
    display: flex;
    align-items: center;
    gap: 10px;
}

.form-icon {
    font-size: 1.4rem;
}

/* Form Grid */
.form-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
    gap: 20px;
    margin-bottom: 25px;
}

.content-layout.centered .form-grid {
    grid-template-columns: repeat(auto-fit, minmax(300px, 1fr));
    gap: 30px;
}

.input-group {
    display: flex;
    flex-direction: column;
}

.input-group.full-width {
    grid-column: 1 / -1;
}

.input-label {
    display: flex;
    align-items: center;
    gap: 5px;
    margin-bottom: 8px;
    font-weight: 500;
    color: #555;
}

.label-text {
    font-size: 0.95rem;
}

.required {
    color: #e74c3c;
    font-weight: bold;
}

.optional {
    color: #7f8c8d;
    font-size: 0.85rem;
    font-weight: normal;
}

.input-wrapper {
    position: relative;
    display: flex;
    align-items: center;
}

.currency-symbol {
    position: absolute;
    left: 15px;
    color: #666;
    font-weight: 500;
    z-index: 1;
}

.input-field {
    width: 100%;
    padding: 15px;
    border: 2px solid #e1e8ed;
    border-radius: 12px;
    font-size: 1rem;
    transition: all 0.3s ease;
    background: #fafbfc;
}

.currency-input {
    padding-left: 45px;
}

.input-field:focus {
    outline: none;
    border-color: #667eea;
    background: white;
    box-shadow: 0 0 0 3px rgba(102, 126, 234, 0.1);
}

.input-hint {
    margin-top: 5px;
    color: #7f8c8d;
    font-size: 0.85rem;
}

/* Buttons */
.button-group {
    display: flex;
    gap: 15px;
    justify-content: center;
    flex-wrap: wrap;
}

.btn {
    padding: 15px 30px;
    border: none;
    border-radius: 12px;
    font-size: 1rem;
    font-weight: 600;
    cursor: pointer;
    transition: all 0.3s ease;
    display: flex;
    align-items: center;
    gap: 8px;
    min-width: 180px;
    justify-content: center;
}

.btn-primary {
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    color: white;
    box-shadow: 0 4px 15px rgba(102, 126, 234, 0.3);
}

.btn-primary:hover {
    transform: translateY(-2px);
    box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
}

.btn-secondary {
    background: #f8f9fa;
    color: #6c757d;
    border: 2px solid #e9ecef;
}

.btn-secondary:hover {
    background: #e9ecef;
    transform: translateY(-1px);
}

.btn-icon {
    font-size: 1.1rem;
}

/* Result Container */
.result-container {
    flex: 1;
    animation: slideInRight 0.5s ease-out;
    min-width: 0;
    overflow: hidden;
}

.content-layout.has-result .result-container {
    max-width: 50%;
}

@keyframes slideInRight {
    from {
        opacity: 0;
        transform: translateX(30px);
    }
    to {
        opacity: 1;
        transform: translateX(0);
    }
}

.result-card {
    background: white;
    border-radius: 20px;
    padding: 25px;
    box-shadow: 0 20px 40px rgba(0,0,0,0.1);
    text-align: center;
    height: 100%;
    display: flex;
    flex-direction: column;
    overflow: hidden;
}

.result-header {
    margin-bottom: 20px;
    flex-shrink: 0;
}

.result-title {
    font-size: 1.5rem;
    font-weight: 600;
    color: #27ae60;
    margin: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 10px;
}

.success-icon {
    font-size: 1.3rem;
}

/* QR Section */
.qr-section {
    margin-bottom: 20px;
    flex-shrink: 0;
}

.qr-container {
    display: inline-block;
    padding: 15px;
    background: white;
    border-radius: 15px;
    box-shadow: 0 8px 25px rgba(0,0,0,0.1);
    border: 3px solid #f8f9fa;
}

.qr-image {
    width: 180px;
    height: 180px;
    border-radius: 8px;
}

/* Info Section */
.info-section {
    flex: 1;
    display: flex;
    flex-direction: column;
}

.info-title {
    font-size: 1.2rem;
    font-weight: 600;
    color: #333;
    margin: 0 0 15px 0;
}

.info-grid {
    display: grid;
    grid-template-columns: 1fr 1fr;
    grid-template-rows: auto auto;
    gap: 10px;
    text-align: left;
    flex: 1;
}

.info-item {
    background: #f8f9fa;
    padding: 10px;
    border-radius: 8px;
    border-left: 4px solid #667eea;
}

.info-label {
    font-weight: 600;
    color: #555;
    display: block;
    margin-bottom: 5px;
    font-size: 0.85rem;
}

.info-value {
    color: #333;
    word-break: break-word;
    font-size: 0.9rem;
}

/* Debug Section */
.debug-section {
    margin-top: 15px;
    flex-shrink: 0;
}

.debug-details {
    background: #f8f9fa;
    border-radius: 8px;
    padding: 10px;
}

.debug-summary {
    font-weight: 600;
    color: #666;
    cursor: pointer;
    display: flex;
    align-items: center;
    gap: 6px;
    list-style: none;
    font-size: 0.85rem;
}

.debug-summary::-webkit-details-marker {
    display: none;
}

.debug-icon {
    font-size: 0.9rem;
}

.debug-content {
    margin-top: 10px;
    padding-top: 10px;
    border-top: 1px solid #e9ecef;
}

.debug-code {
    display: block;
    background: #2d3748;
    color: #e2e8f0;
    padding: 10px;
    border-radius: 6px;
    font-family: 'Courier New', monospace;
    font-size: 0.7rem;
    word-break: break-all;
    line-height: 1.3;
}

/* Popup Styles */
.popup-overlay {
    position: fixed;
    top: 0;
    left: 0;
    width: 100%;
    height: 100%;
    background: rgba(0,0,0,0.5);
    display: flex;
    justify-content: center;
    align-items: center;
    z-index: 1000;
}

.popup {
    background: white;
    border-radius: 15px;
    padding: 0;
    max-width: 400px;
    width: 90%;
    box-shadow: 0 20px 40px rgba(0,0,0,0.2);
    overflow: hidden;
}

.popup.erro {
    border-top: 5px solid #e74c3c;
}

.popup.sucesso {
    border-top: 5px solid #27ae60;
}

.popup-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 20px;
    background: #f8f9fa;
}

.popup-icon {
    font-size: 1.5rem;
}

.popup-close {
    background: none;
    border: none;
    font-size: 1.5rem;
    cursor: pointer;
    color: #666;
    padding: 0;
    width: 30px;
    height: 30px;
    display: flex;
    align-items: center;
    justify-content: center;
    border-radius: 50%;
    transition: background 0.2s;
}

.popup-close:hover {
    background: #e9ecef;
}

.popup-content {
    padding: 20px;
}

.popup-content p {
    margin: 0;
    color: #333;
    font-size: 1rem;
    line-height: 1.5;
}

/* Responsive Design */
@media (max-width: 1200px) {
    .content-layout.has-result {
        flex-direction: column;
        gap: 20px;
        overflow-y: auto;
    }
    
    .content-layout.has-result .form-container,
    .content-layout.has-result .result-container {
        max-width: 100%;
    }
    
    .result-container {
        animation: slideUp 0.5s ease-out;
    }
    
    @keyframes slideUp {
        from {
            opacity: 0;
            transform: translateY(30px);
        }
        to {
            opacity: 1;
            transform: translateY(0);
        }
    }
}

@media (max-width: 768px) {
    .app-container {
        padding: 15px;
    }
    
    .content-layout {
        gap: 15px;
    }
    
    .main-title {
        font-size: 1.8rem;
    }
    
    .subtitle {
        font-size: 1rem;
    }
    
    .form-card {
        padding: 20px;
    }
    
    .form-grid {
        grid-template-columns: 1fr;
        gap: 15px;
    }
    
    .content-layout.centered .form-grid {
        grid-template-columns: 1fr;
        gap: 20px;
    }
    
    .button-group {
        flex-direction: column;
    }
    
    .btn {
        width: 100%;
    }
    
    .qr-image {
        width: 160px;
        height: 160px;
    }
    
    .result-card {
        padding: 20px;
    }
    
    .info-grid {
        grid-template-columns: 1fr;
    }
}

@media (max-width: 480px) {
    .app-container {
        padding: 10px;
    }
    
    .content-layout {
        gap: 10px;
    }
    
    .main-title {
        font-size: 1.6rem;
    }
    
    .logo {
        width: 50px;
        height: 50px;
    }
    
    .form-card {
        padding: 15px;
    }
    
    .content-layout.centered .form-card {
        padding: 20px;
    }
    
    .result-card {
        padding: 15px;
    }
    
    .qr-image {
        width: 140px;
        height: 140px;
    }
}
</style>