<template>
    <v-container class="fill-height d-flex flex-column align-center justify-start pt-10">
        <v-row justify="center">
            <v-col cols="12" class="text-center">
                <h1>🧙‍♂️ Meu Personagem</h1>
            </v-col>
        </v-row>

        <v-row justify="center">
            <v-col cols="8" class="text-center">
                <v-text-field label="Nome do Personagem" size="50" v-model="nomeBusca" @keyup.enter="buscarPersonagem"
                    outlined dense />
            </v-col>
        </v-row>

        <v-row justify="center">
            <v-col cols="auto" class="d-flex" style="gap: 8px;">
                <v-btn color="primary" class="button" @click="buscarPersonagem">
                    Buscar
                </v-btn>
                <v-btn color="secondary" class="button" @click="voltar">
                    Voltar
                </v-btn>
            </v-col>
        </v-row>

        <v-row justify="center" v-if="personagem && personagem.nome">
            <v-col cols="10" md="6">
                <v-card outlined class="pa-4">
                    <v-card-title class="headline">{{ personagem.nome }}</v-card-title>
                    <v-card-text>
                        <p><strong>Dinheiro:</strong> {{ personagem.dinheiro }}
                            <v-icon color="green" class="ml-2 cursor-pointer"
                                @click="abrirDialogDinheiro(true)">mdi-plus-circle</v-icon>
                            <v-icon color="red" class="ml-2 cursor-pointer"
                                @click="abrirDialogDinheiro(false)">mdi-minus-circle</v-icon>
                        </p>

                        <p><strong>Itens:</strong></p>
                        <ul class="indent-list">
                            <li v-for="item in itens" :key="item.id">
                                Nome: {{ item.nome_item }}, Quantidade: {{ item.quantidade }}, Descrição: {{
                                    item.descricao }}
                                <v-icon color="green" class="ml-2 cursor-pointer"
                                    @click="aumentarQuantidade(item)">mdi-plus-circle</v-icon>
                                <v-icon color="red" class="ml-2 cursor-pointer"
                                    @click="diminuirQuantidade(item)">mdi-minus-circle</v-icon>
                            </li>
                        </ul>
                    </v-card-text>
                </v-card>
            </v-col>
        </v-row>

        <!-- Modal de confirmação -->
        <v-dialog v-model="dialog" max-width="400">
            <v-card>
                <v-card-title class="headline">Remover Itens</v-card-title>
                <v-card-text>
                    Deseja realmente remover <strong>{{ itemSelecionado?.nome_item }}</strong>?
                </v-card-text>
                <v-card-actions>
                    <v-spacer />
                    <v-btn text @click="dialog = false">Cancelar</v-btn>
                    <v-btn color="red" @click="removerItem">Remover</v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>

        <!-- Modal de atualização de dinheiro -->
        <v-dialog v-model="dialogDinheiro" max-width="400">
            <v-card>
                <v-card-title class="headline">
                    {{ somandoDinheiro ? "Adicionar Dinheiro" : "Remover Dinheiro" }}
                </v-card-title>
                <v-card-text>
                    <v-text-field v-model.number="valorDinheiro" type="number" :min="1" label="Valor" dense />
                </v-card-text>
                <v-card-actions>
                    <v-spacer />
                    <v-btn text @click="dialogDinheiro = false">Cancelar</v-btn>
                    <v-btn color="primary" @click="alterarDinheiro">
                        {{ somandoDinheiro ? "Adicionar" : "Remover" }}
                    </v-btn>
                </v-card-actions>
            </v-card>
        </v-dialog>
    </v-container>
</template>

<script setup>
import { ref } from 'vue'
import { useRouter } from 'vue-router'
import { buscarPersonagemApi, alterarDinheiroApi, aumentarQuantidadeApi, diminuirQuantidadeApi, removerItemApi } from '../api/api'

const router = useRouter()

const nomeBusca = ref('')
const personagem = ref(null)
const itens = ref([])
const dialog = ref(false)
const itemSelecionado = ref(null)
const quantidadeARemover = ref(1)
const dialogDinheiro = ref(false)
const somandoDinheiro = ref(true)
const valorDinheiro = ref(1)

function voltar() {
    router.push({ name: 'Home' })
}

async function buscarPersonagem() {
    personagem.value = null
    itens.value = []

    try {
        const { personagemData, itensData } = await buscarPersonagemApi(nomeBusca.value)
        if (!personagemData) {
            alert('Personagem não encontrado.')
            return
        }
        personagem.value = personagemData
        itens.value = itensData
    } catch (error) {
        console.error(error)
        alert('Erro ao buscar personagem.')
    }
}

function abrirDialogDinheiro(somar) {
    somandoDinheiro.value = somar
    valorDinheiro.value = 1
    dialogDinheiro.value = true
}

async function alterarDinheiro() {
    if (!personagem.value) return

    if (!Number.isInteger(valorDinheiro.value) || valorDinheiro.value <= 0) {
        alert('Digite um valor inteiro positivo.')
        return
    }

    const valor = somandoDinheiro.value
        ? valorDinheiro.value
        : -valorDinheiro.value

    const novoValor = personagem.value.dinheiro + valor
    if (novoValor < 0) {
        alert('Dinheiro não pode ser negativo.')
        return
    }

    try {
        const dinheiroAtualizado = await alterarDinheiroApi(personagem.value.id, valor)
        personagem.value.dinheiro = dinheiroAtualizado
        dialogDinheiro.value = false
    } catch (error) {
        console.error('Erro ao atualizar dinheiro:', error)
        alert('Erro ao atualizar dinheiro.')
    }
}

async function diminuirQuantidade(item) {
    if (item.quantidade <= 1) {
        itemSelecionado.value = item
        quantidadeARemover.value = 1
        dialog.value = true
    } else {
        try {
            const novaQuantidade = await diminuirQuantidadeApi(item, 1)
            const i = itens.value.findIndex(i => i.id === item.id)
            if (i !== -1) itens.value[i].quantidade = novaQuantidade
        } catch (error) {
            console.error('Erro ao diminuir item:', error)
            alert('Erro ao diminuir item.')
        }
    }
}

async function removerItem() {
    const item = itemSelecionado.value
    const novaQuantidade = item.quantidade - quantidadeARemover.value

    try {
        if (novaQuantidade <= 0) {
            await removerItemApi(item.id)
            itens.value = itens.value.filter(i => i.id !== item.id)
        } else {
            const quantidadeAtualizada = await diminuirQuantidadeApi(item, quantidadeARemover.value)
            const i = itens.value.findIndex(i => i.id === item.id)
            if (i !== -1) itens.value[i].quantidade = quantidadeAtualizada
        }
        dialog.value = false
    } catch (error) {
        console.error('Erro ao remover item:', error)
        alert('Erro ao remover item.')
    }
}

async function aumentarQuantidade(item) {
    try {
        const novaQuantidade = await aumentarQuantidadeApi(item, 1)
        const i = itens.value.findIndex(i => i.id === item.id)
        if (i !== -1) itens.value[i].quantidade = novaQuantidade
    } catch (error) {
        console.error('Erro ao aumentar item:', error)
        alert('Erro ao aumentar item.')
    }
}
</script>

<style scoped>
.button {
    min-width: 140px;
    min-height: 40px;
    font-size: 0.95rem;
    text-transform: none;
    padding: 6px 12px;
}

.indent-list {
    padding-left: 1.5rem;
    list-style-type: disc;
    margin-top: 0;
    margin-bottom: 0;
}

.v-row {
    flex: none;
}

.v-row>.v-col {
    flex-grow: 0;
    flex-shrink: 0;
    flex: none;
}
</style>