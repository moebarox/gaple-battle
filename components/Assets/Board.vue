<template>
  <div
    class="absolute top-16 left-8 right-8 bottom-28 bg-[#27aa48] rounded-full border-8 border-[#22933f] drop-shadow-2xl shadow-inner before:absolute before:top-4 before:left-4 before:right-4 before:bottom-4 before:rounded-full before:border-2 before:border-[#22933f] before:z-[-1]"
  >
    <div ref="innerBoard" class="absolute w-5/6 h-5/6 left-1/2 top-1/2 -translate-x-1/2 -translate-y-1/2">
      <div
        v-if="showHeadPlaceholder || board.length === 0"
        class="absolute rounded-md bg-[#279a43] border-[1px] border-dashed border-[#22933f] w-[30px] h-[60px] z-10"
        :class="{
          'cursor-pointer': canSelectPosition,
        }"
        :style="generatePositionStyle(BOARD_POSITION.head)"
        @click="handleSelect(BOARD_POSITION.head)"
      ></div>
      <AssetsDomino
        v-for="card in board"
        :key="card"
        :card="card"
        :width="cardSize / 2"
        class="absolute"
        :style="generateCardStyle(card)"
      />
      <div
        v-if="showTailPlaceholder && board.length > 0"
        class="absolute rounded-md bg-[#279a43] border-[1px] border-dashed border-[#22933f] w-[30px] h-[60px] z-10"
        :class="{
          'cursor-pointer': canSelectPosition,
        }"
        :style="generatePositionStyle(BOARD_POSITION.tail)"
        @click="handleSelect(BOARD_POSITION.tail)"
      ></div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { BOARD_POSITION, CARD_DIRECTION } from '#imports'

const { board, totalRows, cardPerRow, getRow, getColumn, getDirection, getRelativeIdx, getCardPosition } = useBoard()

const props = defineProps<{
  showHeadPlaceholder?: boolean
  showTailPlaceholder?: boolean
  canSelectPosition?: boolean
}>()

const emits = defineEmits<{
  (e: 'select', position: BOARD_POSITION): void
}>()

const innerBoard = ref<HTMLDivElement | null>(null)
const innerBoardWidth = ref(0)
const innerBoardHeight = ref(0)
const cardSize = ref(60)

const topPadding = computed(() => (innerBoardHeight.value - cardSize.value * totalRows.value) / 2)
const leftPadding = computed(() => (innerBoardWidth.value - cardSize.value * cardPerRow.value) / 2)

const generatePositionStyle = (position: BOARD_POSITION) => {
  const relativeIdx = position === BOARD_POSITION.head ? getRelativeIdx(0) - 1 : getRelativeIdx(board.value.length)

  const row = getRow(relativeIdx)
  const col = getColumn(relativeIdx, row)
  const direction = getDirection(row)

  return generateStyle(row, col, direction)
}

const generateCardStyle = (card: string) => {
  const isReversed = isCardReversed(card)
  const { row, col, direction } = getCardPosition(card)

  return generateStyle(row, col, direction, isReversed)
}

const generateStyle = (row: number, col: number, direction: CARD_DIRECTION, isReversed?: boolean) => {
  const style: Record<string, string> = {}

  let top = topPadding.value + row * cardSize.value
  let left = leftPadding.value + col * cardSize.value

  if (direction === CARD_DIRECTION.left) {
    if (col === 0) {
      top += cardSize.value / 4
      left += cardSize.value / 4
    }
  } else {
    left += cardSize.value / 4

    if (col === cardPerRow.value - 1) {
      top += cardSize.value / 4
    } else {
      left += cardSize.value / 4
    }
  }

  style.top = `${top}px`
  style.left = `${left}px`

  switch (true) {
    case direction === CARD_DIRECTION.right && isReversed && col === cardPerRow.value - 1:
    case direction === CARD_DIRECTION.left && isReversed && col === 0:
      style.transform = 'rotate(180deg)'
      break
    case direction === CARD_DIRECTION.right && col === cardPerRow.value - 1:
    case direction === CARD_DIRECTION.left && col === 0:
      break
    case direction === CARD_DIRECTION.right && isReversed:
    case direction === CARD_DIRECTION.left && !isReversed:
      style.transform = 'rotate(90deg)'
      break
    case direction === CARD_DIRECTION.right && !isReversed:
    case direction === CARD_DIRECTION.left && isReversed:
      style.transform = 'rotate(-90deg)'
      break
  }

  return style
}

const isCardReversed = card => card.at(0) > card.at(2)

const handleSelect = position => {
  if (!props.canSelectPosition) return

  emits('select', position)
}

const calculateBoardSize = () => {
  if (!innerBoard.value) {
    return
  }

  cardPerRow.value = Math.floor(innerBoard.value.offsetWidth / cardSize.value)
  innerBoardWidth.value = innerBoard.value.offsetWidth
  innerBoardHeight.value = innerBoard.value.offsetHeight
}

onMounted(() => {
  window.addEventListener('resize', calculateBoardSize)
  calculateBoardSize()
})
</script>
