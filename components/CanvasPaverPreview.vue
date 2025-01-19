<template>
    <div ref="containerRef" style="width: 100%; height: 100%;">
        <canvas ref="canvasRef" style="display: block; width: 100%; height: 100%;"></canvas>
    </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount, watch } from 'vue'

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

const containerRef = ref<HTMLDivElement | null>(null)
const canvasRef = ref<HTMLCanvasElement | null>(null)

/**
 * Redimensionne le canvas à la taille du conteneur
 * et redessine le motif.
 */
function resizeCanvasAndDraw() {
    if (!containerRef.value || !canvasRef.value) return

    const container = containerRef.value
    const canvas = canvasRef.value

    // Mesurer le conteneur
    const rect = container.getBoundingClientRect()
    const dpr = window.devicePixelRatio || 1

    // Ajuster la résolution interne du canvas
    canvas.width = rect.width * dpr
    canvas.height = rect.height * dpr

    // Corriger l'échelle si dessin 2D
    const ctx = canvas.getContext('2d')
    if (ctx) {
        ctx.scale(dpr, dpr)
    }

    // Appeler la fonction de dessin
    draw()
}

/**
 * Fonction principale de dessin :
 * - Calcule l'échelle (pixels par mètre).
 * - En fonction de "pattern", appelle le sous-algorithme.
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
        pattern = 'straight'
    } = props

    // Vérif
    if (terraceWidth <= 0 || terraceHeight <= 0 || paverWidth <= 0 || paverLength <= 0) {
        return
    }

    // Calcul d'échelle
    const padding = 20
    const availW = canvas.width - 2 * padding
    const availH = canvas.height - 2 * padding
    const scaleX = availW / terraceWidth
    const scaleY = availH / terraceHeight
    const scale = Math.min(scaleX, scaleY)

    // Contour en rouge
    const xMin = padding
    const yMin = padding
    const xMax = xMin + terraceWidth * scale
    const yMax = yMin + terraceHeight * scale

    ctx.strokeStyle = 'red'
    ctx.lineWidth = 2
    ctx.strokeRect(xMin, yMin, terraceWidth * scale, terraceHeight * scale)

    // Dessin en fonction du motif
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
            console.warn(`Motif inconnu : ${pattern}`)
            break
    }
}

/*--- Variantes de motifs (straight, staggered, herringbone) ---*/

function drawStraight(
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
        for (let col = 0; col < nbX; col++) {
            const x = xMin + col * pWidthPx
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
        // Sur rangées impaires : offset négatif
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

    const blockSizeX = pLengthPx + pWidthPx
    const blockSizeY = pLengthPx + pWidthPx

    const nbBlockX = Math.ceil(terraceWidth * scale / blockSizeX)
    const nbBlockY = Math.ceil(terraceHeight * scale / blockSizeY)

    for (let row = 0; row < nbBlockY; row++) {
        for (let col = 0; col < nbBlockX; col++) {
            const baseX = xMin + col * blockSizeX
            const baseY = yMin + row * blockSizeY

            // Pavé A (horizontal)
            drawClippedRect(ctx, baseX, baseY, pLengthPx, pWidthPx, xMin, yMin, xMax, yMax)
            // Pavé B (vertical)
            drawClippedRect(ctx, baseX + pLengthPx, baseY, pWidthPx, pLengthPx, xMin, yMin, xMax, yMax)
        }
    }
}

function drawClippedRect(
    ctx: CanvasRenderingContext2D,
    x: number, y: number,
    width: number, height: number,
    xMin: number, yMin: number,
    xMax: number, yMax: number
) {
    const xRight = x + width
    const yBottom = y + height

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

/*-----------------------------------
    Hooks du cycle de vie
-----------------------------------*/
onMounted(() => {
    // 1) Au montage, on redimensionne et on dessine
    resizeCanvasAndDraw()

    // 2) On écoute le resize de la fenêtre
    window.addEventListener('resize', resizeCanvasAndDraw)
})

onBeforeUnmount(() => {
    window.removeEventListener('resize', resizeCanvasAndDraw)
})

// 3) Quand les props changent (taille terrasse, pavés, motif), on redessine
watch(
    () => [props.terraceWidth, props.terraceHeight, props.paverWidth, props.paverLength, props.pattern],
    () => {
        resizeCanvasAndDraw()
    }
)
</script>

<style scoped>
/* Ajustez le style selon vos besoins */
</style>