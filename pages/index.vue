<template>
    <v-app>
        <v-main>
            <v-container class="py-10">
                <v-row justify="center">
                    <v-col cols="12" md="8">
                        <v-card class="pa-6">
                            <v-card-title>Calcul du nombre de pavés pour la terrasse</v-card-title>
                            <v-card-subtitle>
                                Saisissez les informations ci-dessous pour estimer la quantité.
                            </v-card-subtitle>

                            <v-card-text>
                                <v-form v-model="formIsValid" ref="formRef">
                                    <!-- Longueur de la terrasse (m) -->
                                    <v-text-field v-model.number="terraceLength" label="Longueur de la terrasse (m)"
                                        type="number" :rules="[v => !!v || 'Requis']" required></v-text-field>

                                    <!-- Largeur de la terrasse (m) -->
                                    <v-text-field v-model.number="terraceWidth" label="Largeur de la terrasse (m)"
                                        type="number" :rules="[v => !!v || 'Requis']" required></v-text-field>

                                    <!-- Longueur d'un pavé (m) -->
                                    <v-text-field v-model.number="paverLength" label="Longueur d'un pavé (m)"
                                        type="number" :rules="[v => !!v || 'Requis']" required></v-text-field>

                                    <!-- Largeur d'un pavé (m) -->
                                    <v-text-field v-model.number="paverWidth" label="Largeur d'un pavé (m)"
                                        type="number" :rules="[v => !!v || 'Requis']" required></v-text-field>

                                    <!-- Prix unitaire d'un pavé -->
                                    <v-text-field v-model.number="paverPrice" label="Prix unitaire d'un pavé (€)"
                                        type="number" :rules="[v => !!v || 'Requis']" required></v-text-field>

                                    <v-btn color="primary" class="mt-4" @click="calculate">
                                        Calculer
                                    </v-btn>
                                </v-form>

                                <v-alert v-if="resultMessage" type="info" class="mt-6">
                                    {{ resultMessage }}
                                </v-alert>
                            </v-card-text>
                        </v-card>

                        <!-- Choix du motif de pose -->
                        <v-select v-model="selectedPattern" :items="['straight', 'staggered']"
                            label="Choisissez le motif de pose" class="pt-10" />
                    </v-col>

                    <!-- Aperçu visuel via CanvasPaverPreview -->
                    <CanvasPaverPreview :terraceWidth="terraceWidth" :terraceHeight="terraceLength"
                        :paverWidth="paverWidth" :paverLength="paverLength" :pattern="selectedPattern" />

                </v-row>
            </v-container>
        </v-main>
    </v-app>
</template>

<script setup lang="ts">
import { ref } from 'vue'
import CanvasPaverPreview from '~/components/CanvasPaverPreview.vue'

// Formulaire
const formRef = ref<any>(null)
const formIsValid = ref<boolean>(true)

// Terrasse (en mètres)
const terraceLength = ref<number>(5) // Longueur
const terraceWidth = ref<number>(4)  // Largeur

// Pavé (en mètres)
const paverLength = ref<number>(0)
const paverWidth = ref<number>(0)

// Prix unitaire du pavé
const paverPrice = ref<number>(0)

// Motif de pose
const selectedPattern = ref<'straight' | 'staggered'>('straight')

// Résultat à afficher
const resultMessage = ref<string | null>(null)

/**
 * Calcul du nombre de pavés :
 * 1) Surface de la terrasse = terraceLength × terraceWidth
 * 2) Surface d’un pavé = paverLength × paverWidth
 * 3) Nombre de pavés = (Surface terrasse) / (Surface pavé)
 * 4) Arrondi au supérieur
 */
function calculate() {
    if (!formRef.value.validate()) {
        return
    }

    // Calcul de la surface de la terrasse
    const surface = terraceLength.value * terraceWidth.value
    if (surface <= 0) {
        resultMessage.value = 'Veuillez saisir des dimensions de terrasse valides.'
        return
    }

    // Surface d'un pavé
    const surfacePaver = paverLength.value * paverWidth.value
    if (surfacePaver <= 0) {
        resultMessage.value = 'Veuillez saisir des dimensions de pavé valides.'
        return
    }

    // Nombre de pavés
    const rawCount = surface / surfacePaver
    const exactCount = Math.ceil(rawCount)

    // Coût total
    const totalCost = exactCount * paverPrice.value

    // Message qui inclut la superficie
    resultMessage.value = `
    Superficie totale : ${surface.toFixed(2)} m²
    | Nombre de pavés nécessaires : ${exactCount}
    | Coût total estimé : ${totalCost.toFixed(2)} €
  `
}

</script>