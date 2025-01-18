<template>
    <div>
        <canvas ref="canvasRef" width="800" height="600"></canvas>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted, watch } from 'vue'

/**
 * Props du composant :
 * - terraceWidth, terraceHeight : dimensions de la terrasse (m)
 * - paverWidth, paverLength     : dimensions du pavé (m)
 * - pattern                     : "straight", "staggered", ou "herringbone"
 */
interface Props {
    terraceWidth: number
    terraceHeight: number
    paverWidth: number
    paverLength: number
    pattern?: string
}

const props = defineProps<Props>()

const canvasRef = ref<HTMLCanvasElement | null>(null)

// Dessin au montage et à chaque changement de props
onMounted(draw)
watch(
    () => [props.terraceWidth, props.terraceHeight, props.paverWidth, props.paverLength, props.pattern],
    draw
)

/**
 * Fonction principale de dessin :
 * - Calcule l'échelle (pixels par mètre).
 * - En fonction de "pattern", appelle le sous-algorithme correspondant.
 */
function draw() {
    const canvas = canvasRef.value
    if (!canvas) return

    const ctx = canvas.getContext('2d')
    if (!ctx) return

    // Nettoyer le canvas
    ctx.clearRect(0, 0, canvas.width, canvas.height)

    const {
        terraceWidth,
        terraceHeight,
        paverWidth,
        paverLength,
        pattern = 'straight'  // default
    } = props

    // Vérif des entrées
    if (terraceWidth <= 0 || terraceHeight <= 0 || paverWidth <= 0 || paverLength <= 0) {
        return
    }

    // Dimensions et échelle
    const padding = 20
    const availW = canvas.width - 2 * padding
    const availH = canvas.height - 2 * padding
    const scaleX = availW / terraceWidth
    const scaleY = availH / terraceHeight
    const scale = Math.min(scaleX, scaleY)

    // Tracer le contour de la terrasse en rouge
    const xMin = padding
    const yMin = padding
    const xMax = xMin + terraceWidth * scale
    const yMax = yMin + terraceHeight * scale

    ctx.strokeStyle = 'red'
    ctx.lineWidth = 2
    ctx.strokeRect(xMin, yMin, terraceWidth * scale, terraceHeight * scale)

    // Selon le motif, on appelle un sous-algo :
    switch (pattern) {
        case 'straight':
            drawStraight(ctx, xMin, yMin, xMax, yMax, scale)
            break
        case 'staggered':
            drawStaggered(ctx, xMin, yMin, xMax, yMax, scale)
            break
        case 'herringbone':
            drawHerringbone(ctx, xMin, yMin, xMax, yMax, scale)
            break
        default:
            // Si pattern inconnu, on ne dessine rien
            console.warn(`Motif inconnu : ${pattern}`)
            break
    }
}

/**
 * 1) Motif "straight" (droit) :
 *    - Pas de décalage, pas de rotation, on place les pavés en rangées/colonnes.
 */
function drawStraight(
    ctx: CanvasRenderingContext2D,
    xMin: number, yMin: number, xMax: number, yMax: number,
    scale: number
) {
    const { paverWidth, paverLength, terraceWidth, terraceHeight } = props

    const pWidthPx = paverWidth * scale
    const pLengthPx = paverLength * scale

    // Nombre de pavés par direction
    const nbX = Math.ceil(terraceWidth / paverWidth)
    const nbY = Math.ceil(terraceHeight / paverLength)

    ctx.fillStyle = '#aabbaa'
    ctx.strokeStyle = '#333'
    ctx.lineWidth = 1

    for (let row = 0; row < nbY; row++) {
        for (let col = 0; col < nbX; col++) {
            const x = xMin + col * pWidthPx
            const y = yMin + row * pLengthPx

            // Rognage
            const xRight = x + pWidthPx
            const yBottom = y + pLengthPx
            if (xRight <= xMin || x >= xMax) continue
            if (yBottom <= yMin || y >= yMax) continue

            const clippedX = Math.max(x, xMin)
            const clippedWidth = Math.min(xRight, xMax) - clippedX
            const clippedY = Math.max(y, yMin)
            const clippedHeight = Math.min(yBottom, yMax) - clippedY

            ctx.beginPath()
            ctx.rect(clippedX, clippedY, clippedWidth, clippedHeight)
            ctx.fill()
            ctx.stroke()
        }
    }
}

/**
 * 2) Motif "staggered" (quinconce) :
 *    - Sur les rangées impaires, on décale le début d'un demi-pavé (négatif ou positif).
 */
function drawStaggered(
    ctx: CanvasRenderingContext2D,
    xMin: number, yMin: number, xMax: number, yMax: number,
    scale: number
) {
    const { paverWidth, paverLength, terraceWidth, terraceHeight } = props

    const pWidthPx = paverWidth * scale
    const pLengthPx = paverLength * scale

    const nbX = Math.ceil(terraceWidth / paverWidth)
    const nbY = Math.ceil(terraceHeight / paverLength)

    ctx.fillStyle = '#aabbaa'
    ctx.strokeStyle = '#333'
    ctx.lineWidth = 1

    for (let row = 0; row < nbY; row++) {
        // Choix : offset négatif pour combler le bord gauche
        // => sur rangées impaires : offset = -pWidthPx/2
        // sur rangées paires : offset = 0
        const offsetX = (row % 2 !== 0) ? -pWidthPx / 2 : 0

        for (let col = 0; col < nbX; col++) {
            const x = xMin + col * pWidthPx + offsetX
            const y = yMin + row * pLengthPx

            const xRight = x + pWidthPx
            const yBottom = y + pLengthPx
            if (xRight <= xMin || x >= xMax) continue
            if (yBottom <= yMin || y >= yMax) continue

            const clippedX = Math.max(x, xMin)
            const clippedWidth = Math.min(xRight, xMax) - clippedX
            const clippedY = Math.max(y, yMin)
            const clippedHeight = Math.min(yBottom, yMax) - clippedY

            ctx.beginPath()
            ctx.rect(clippedX, clippedY, clippedWidth, clippedHeight)
            ctx.fill()
            ctx.stroke()
        }
    }
}

/**
 * 3) Motif "herringbone" (bâtons rompus basique) :
 *    - On dessine des blocs de 2 pavés :
 *        - Pavé A horizontal
 *        - Pavé B vertical
 *      formant un "L".
 *    - On répète ce bloc le long d'une rangée, puis on descend sur la suivante.
 *    - NB : c'est un motif simplifié, angle = 90° entre pavés,
 *           sans rotation à 45° (pour un vrai herringbone incliné, il faudrait plus de maths).
 */
function drawHerringbone(
    ctx: CanvasRenderingContext2D,
    xMin: number, yMin: number, xMax: number, yMax: number,
    scale: number
) {
    const { paverWidth, paverLength, terraceWidth, terraceHeight } = props

    const pWidthPx = paverWidth * scale
    const pLengthPx = paverLength * scale

    ctx.fillStyle = '#aabbaa'
    ctx.strokeStyle = '#333'
    ctx.lineWidth = 1

    // Taille d'un "bloc" =  paverLength * paverWidth ...
    // En fait, un bloc "L" fait environ paverLength + paverWidth en largeur, paverLength + paverWidth en hauteur.
    // On va simplifier en se disant qu'un bloc est de la taille max (pLengthPx + pWidthPx) dans chaque direction
    // pour être sûr de couvrir l'encombrement.
    const blockSizeX = pLengthPx + pWidthPx
    const blockSizeY = pLengthPx + pWidthPx

    // Combien de blocs en X et Y ?
    const nbBlockX = Math.ceil(terraceWidth * scale / blockSizeX)
    const nbBlockY = Math.ceil(terraceHeight * scale / blockSizeY)

    for (let row = 0; row < nbBlockY; row++) {
        for (let col = 0; col < nbBlockX; col++) {
            // Position "bloc"
            const baseX = xMin + col * blockSizeX
            const baseY = yMin + row * blockSizeY

            // 1) Pavé A : horizontal, posé dans la partie haute du bloc
            //    Largeur = pLengthPx, Hauteur = pWidthPx
            //    => (x, y) = (baseX, baseY)
            drawClippedRect(ctx, baseX, baseY, pLengthPx, pWidthPx, xMin, yMin, xMax, yMax)

            // 2) Pavé B : vertical, collé au bout du pavé A
            //    => (x, y) = (baseX + pLengthPx, baseY)
            //    Largeur = pWidthPx, Hauteur = pLengthPx
            drawClippedRect(ctx, baseX + pLengthPx, baseY, pWidthPx, pLengthPx, xMin, yMin, xMax, yMax)
        }
    }
}

/**
 * Petite fonction utilitaire pour dessiner un rectangle rogné dans (xMin,yMin,xMax,yMax).
 */
function drawClippedRect(
    ctx: CanvasRenderingContext2D,
    x: number, y: number,
    width: number, height: number,
    xMin: number, yMin: number,
    xMax: number, yMax: number
) {
    const xRight = x + width
    const yBottom = y + height

    // Vérif si hors zone
    if (xRight <= xMin || x >= xMax) return
    if (yBottom <= yMin || y >= yMax) return

    const clippedX = Math.max(x, xMin)
    const clippedWidth = Math.min(xRight, xMax) - clippedX

    const clippedY = Math.max(y, yMin)
    const clippedHeight = Math.min(yBottom, yMax) - clippedY

    ctx.beginPath()
    ctx.rect(clippedX, clippedY, clippedWidth, clippedHeight)
    ctx.fill()
    ctx.stroke()
}
</script>

<style scoped>
/* Style libre, si besoin */
</style>