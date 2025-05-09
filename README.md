# Tumpermonkey
Código de digitador de links

// ==UserScript==
// @name Auto-Navigator e Impressor de URLs Alura (v2)
// @namespace http://tampermonkey.net/
// @version 1.3
// @description Navega automaticamente por uma lista de URLs e aciona a impressão (requer interação para salvar como PDF)
// @author Seu Nome
// @match https://cursos.alura.com.br/corp/company/17745/user/*
// @grant GM_getValue
// @grant GM_setValue
// @grant GM_registerMenuCommand
// @run-at document-idle
// ==/UserScript==

(async function() {
    'use strict';

    // Lista completa das URLs a serem navegadas e impressas
    const URLS = [
        'https://cursos.alura.com.br/corp/company/17745/user/rg000000000sp',
        // escreva os seus links aqui como no exemplo acima...
    ];

    // Chave para armazenar o índice atual no Tampermonkey Storage
    const STORAGE_KEY = 'currentUrlIndex';
    // Atraso para permitir a interação com a caixa de diálogo de impressão (em milissegundos)
    const PRINT_DIALOG_DELAY = 5000; // Ajuste conforme necessário
     // Atraso para dar tempo da página renderizar antes de imprimir
    const RENDER_DELAY = 3000; // Ajuste se a página demorar para exibir o conteúdo

    // Função a ser executada após o carregamento e ociosidade de CADA página
    async function afterPageLoad() {
        console.log("afterPageLoad triggered.");
        const currentIndex = await GM_getValue(STORAGE_KEY, 0);
        const currentUrl = window.location.href;

        console.log(`Current Index: ${currentIndex}, Current URL: ${currentUrl}`);

        // Se o índice for 0, significa que ainda não começamos a navegação sequencial.
        // O script apenas aguarda o comando do menu.
        if (currentIndex === 0) {
            console.log("Índice é 0. Aguardando comando do menu para iniciar.");
            return;
        }

        // Se o índice for maior que 0, verificamos se a URL atual é a que esperávamos
        // carregar com base no índice salvo ANTES da navegação.
        // O índice salvo (currentIndex) aponta para a *próxima* URL a ser visitada.
        // Portanto, a URL atual deve ser a de índice (currentIndex - 1) na lista.
        if (currentIndex > 0 && currentIndex <= URLS.length) {
            const expectedUrl = URLS[currentIndex - 1]; // A URL que acabamos de carregar

            // Usamos startsWith para sermos um pouco mais flexíveis com possíveis parâmetros na URL
            if (currentUrl.startsWith(expectedUrl)) {
                console.log(`Página carregada corresponde à URL esperada no índice ${currentIndex - 1}. Acionando impressão.`);

                // Espera um pouco para garantir que o conteúdo dinâmico esteja renderizado
                await new Promise(resolve => setTimeout(resolve, RENDER_DELAY));

                console.log("Acionando window.print()...");
                window.print();

                // Espera para dar tempo do usuário interagir com a caixa de diálogo de impressão
                await new Promise(resolve => setTimeout(resolve, PRINT_DIALOG_DELAY));

                // Navega para a próxima URL, se houver
                if (currentIndex < URLS.length) {
                    const nextUrl = URLS[currentIndex];
                    // Incrementa o índice *antes* de navegar para que a próxima página saiba seu número
                    await GM_setValue(STORAGE_KEY, currentIndex + 1);
                    console.log(`Navegando para a próxima URL ${currentIndex + 1}/${URLS.length}: ${nextUrl}`);
                    window.location.href = nextUrl;
                } else {
                    // Se chegamos ao fim da lista
                    console.log('Todas as URLs foram processadas!');
                    alert('Todas as URLs foram processadas!');
                    await GM_setValue(STORAGE_KEY, 0); // Reinicia o índice
                }
            } else {
                 console.warn(`Página carregada (${currentUrl}) não corresponde à URL esperada (${expectedUrl}) para o índice ${currentIndex}. O processo pode ter sido interrompido ou reiniciado manualmente. Script aguardando.`);
                 // Se a página carregada não for a esperada, o script pausa e aguarda o comando do menu.
            }
        } else if (currentIndex > URLS.length) {
             console.warn(`Índice salvo (${currentIndex}) está fora dos limites da lista de URLs. Reiniciando índice para 0.`);
             await GM_setValue(STORAGE_KEY, 0); // Reinicia se o índice estiver inválido
        }
    }

    // O comando do menu simplesmente inicia o processo navegando para a primeira URL
    GM_registerMenuCommand('Iniciar Navegação e Impressão', async () => {
        const currentIndex = await GM_getValue(STORAGE_KEY, 0);
        if (currentIndex === 0) {
            console.log("Iniciando navegação a partir do menu. Navegando para a primeira URL.");
            // Define o índice para 1 ANTES de navegar para a URL de índice 0.
            // Quando a página 0 carregar, afterPageLoad verá o índice 1 e processará a URL 0.
            await GM_setValue(STORAGE_KEY, 1);
            window.location.href = URLS[0];
        } else {
            alert(`A navegação automática já está em andamento ou pausada no índice ${currentIndex}. Use "Reiniciar Índice" para começar de novo.`);
             console.log(`Comando de início ignorado. Processo em andamento no índice ${currentIndex}.`);
        }
    });

    GM_registerMenuCommand('Reiniciar Índice', async () => {
        console.log('Comando Reiniciar Índice acionado. Resetando para 0.');
        await GM_setValue(STORAGE_KEY, 0);
        alert('Índice de navegação reiniciado para 0.');
    });

    // Adiciona um listener para executar afterPageLoad toda vez que uma página carregar
    // Usamos 'DOMContentLoaded' e 'load' para aumentar a chance de pegar o carregamento.
    window.addEventListener('DOMContentLoaded', afterPageLoad);
    window.addEventListener('load', afterPageLoad);


})();
