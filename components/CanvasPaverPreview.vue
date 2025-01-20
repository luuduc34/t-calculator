<template>
    <div ref="containerRef" style="width: 100%; height: 100%;">
        <canvas ref="canvasRef" style="display: flex; width: 100%; height: 100%;"></canvas>
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

// 1) Au montage : on met à la taille du conteneur, on dessine
// 2) On écoute le resize window pour s'adapter en direct
onMounted(() => {
    resizeCanvasAndDraw()
    window.addEventListener('resize', resizeCanvasAndDraw)
})

// On se désabonne du resize quand le composant se démonte
onBeforeUnmount(() => {
    window.removeEventListener('resize', resizeCanvasAndDraw)
})

// 3) À chaque changement de props, on redimensionne + redessine
watch(
    () => [props.terraceWidth, props.terraceHeight, props.paverWidth, props.paverLength, props.pattern],
    () => {
        resizeCanvasAndDraw()
    }
)

// Fonction qui redimensionne le canvas (sa résolution interne) 
// en fonction de la taille du conteneur, puis appelle draw().
function resizeCanvasAndDraw() {
    if (!containerRef.value || !canvasRef.value) return

    const container = containerRef.value
    const canvas = canvasRef.value

    // Mesurer le conteneur
    const rect = container.getBoundingClientRect()

    // Gérer le ratio pixel (pour éviter le flou sur écrans HD/Retina)
    const dpr = window.devicePixelRatio || 1

    // Ajuster la résolution interne du canvas
    canvas.width = rect.width * dpr
    canvas.height = rect.height * dpr

    // Exemple : si on dessine en 2D, on peut corriger l'échelle
    const ctx = canvas.getContext('2d')
    if (ctx) {
        ctx.scale(dpr, dpr)
    }

    // Appeler la fonction de dessin
    draw()
}

// Fonction qui ajuste la taille interne du canvas à celle du conteneur
function resizeCanvas() {
    if (!containerRef.value || !canvasRef.value) return

    const rect = containerRef.value.getBoundingClientRect()
    const devicePixelRatio = window.devicePixelRatio || 1

    // Ajuster la résolution interne (en pixels) du canvas
    canvasRef.value.width = rect.width * devicePixelRatio
    canvasRef.value.height = rect.height * devicePixelRatio

    // Facultatif : si vous dessinez quelque chose, redessinez tout maintenant
    const ctx = canvasRef.value.getContext('2d')
    if (ctx) {
        // Exemple de test : un rectangle occupant toute la zone
        ctx.scale(devicePixelRatio, devicePixelRatio)  // pour “corriger” l’échelle
        ctx.fillStyle = '#aabbaa'
        ctx.fillRect(0, 0, rect.width, rect.height)
    }
}

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
    const availW = canvas.width
    const availH = canvas.height
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
