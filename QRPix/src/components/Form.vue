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

function mostrarMensagem(mensagem: string, tipo: string='erro') {
    mostrarPopUp.value = true;
    mensagemPopup.value = mensagem;
    tipoPopup.value = tipo;
}

setTimeout(() => {
    mostrarPopUp.value = false;
}, 3000);


</script>

<template>

</template>

<style scoped></style>