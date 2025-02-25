<template>
  <div class="relative flex justify-center items-center h-dvh overflow-hidden">
    <AssetsBoard
      :show-head-placeholder="!isMatchOver && isPlayerTurn && possiblyPlacedOnHead"
      :show-tail-placeholder="!isMatchOver && isPlayerTurn && possiblyPlacedOnTail"
      :can-select-position="!isMatchOver && isPlayerTurn && turnState === TURN_STATE.selectPosition"
      @select="handleSelectPosition"
    />

    <AssetsPlayerInfo
      v-for="(player, idx) in otherPlayers"
      :key="player.id"
      show-cards
      :player="player"
      :position="idx"
      :is-reveal-card="isMatchOver"
      :is-highlighted="!isMatchOver && match?.state?.turn === player.id"
      :is-rt="match?.state?.rt === player.id"
    />

    <div class="absolute text-center bottom-0 w-full">
      <div class="flex flex-col items-center">
        <div class="flex justify-center grow gap-3 p-2">
          <AssetsDomino
            v-for="card in currentPlayer.cards"
            :key="card"
            :card="card"
            :width="cardSize"
            :is-selectable="!isMatchOver && isSelectable(card)"
            :is-disabled="!isMatchOver && isPlayerTurn && !selectableCards.includes(card)"
            :class="{
              '-translate-y-4': card === selectedCard,
            }"
            @select="handleSelectCard"
          />
        </div>
        <AssetsPlayerInfo
          :player="currentPlayer"
          :position="PLAYER_INFO_POSITION.bottom"
          :is-highlighted="!isMatchOver && match?.state?.turn === currentPlayer.id"
          :is-rt="match?.state?.rt === currentPlayer.id"
        />
      </div>
    </div>

    <UButton
      v-if="isMatchOver && isRoomMaster"
      size="lg"
      color="primary"
      variant="solid"
      class="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 drop-shadow-2xl"
      @click="newRound"
    >
      {{ $t('action.newRound') }}
    </UButton>
  </div>
</template>

<script setup lang="ts">
import { doc, onSnapshot, updateDoc, increment, type DocumentData } from 'firebase/firestore'

import {
  TURN_STATE,
  BOARD_POSITION,
  FORBIDDEN_FIRST_TURN_CARDS,
  SKIP_POINT,
  GAPLE_POINT,
  POLDAN_POINT,
  DEFAULT_TOAST_TIMEOUT,
  MEDIUM_TOAST_TIMEOUT,
} from '#imports'

const route = useRoute()
const { $db } = useNuxtApp()
const { t } = useI18n()
const { user } = useUser()
const toast = useToast()
const {
  match,
  players,
  state,
  board,
  head,
  tail,
  playerIdx,
  currentPlayer,
  nextPlayer,
  rtPlayer,
  isRoomMaster,
  isPlayerTurn,
  isMatchOver,
  isMatchDraw,
} = useMatch()

const matchId = route.params.id as string

const turnState = ref<TURN_STATE>(TURN_STATE.pickCard)
const selectedCard = ref('')
const unsubGameplay = ref()
const cardSize = ref(40)

const otherPlayers = computed<TMatchPlayer[]>(() => {
  const idx = playerIdx.value
  return players.value.slice(idx + 1, players.value.length).concat(players.value.slice(0, idx))
})

const selectableCards = computed<string[]>(() => {
  if (board.value.length === 0) {
    if (state.value.firstTurnCard) {
      return [state.value.firstTurnCard]
    } else {
      return currentPlayer.value.cards?.filter(c => !FORBIDDEN_FIRST_TURN_CARDS.includes(c)) ?? []
    }
  }

  return getPossibleCards(currentPlayer.value.cards, head.value, tail.value)
})

const possiblyPlacedOnHead = computed(() => selectableCards.value.some(c => c.includes(head.value)))
const possiblyPlacedOnTail = computed(() => selectableCards.value.some(c => c.includes(tail.value)))

const isSelectable = (card: string) => isPlayerTurn.value && selectableCards.value.includes(card)

const handleTurn = async doc => {
  match.value = doc.data()

  if (isMatchOver.value) {
    toast.add({
      title: t('notification.gameOver'),
      timeout: MEDIUM_TOAST_TIMEOUT,
    })

    if (rtPlayer.value) {
      toast.add({
        title: t('notification.newRT', { name: rtPlayer.value.name, point: rtPlayer.value.penalty }),
        timeout: MEDIUM_TOAST_TIMEOUT,
      })
    }

    return
  }

  if (!isPlayerTurn.value) {
    return
  }

  if (selectableCards.value.length === 0) {
    await new Promise(resolve => setTimeout(resolve, 1000))

    toast.add({
      title: t('notification.skipTurn'),
      timeout: MEDIUM_TOAST_TIMEOUT,
      click: () => {
        skipTurn()
      },
      callback: () => {
        skipTurn()
      },
    })

    return
  }

  toast.add({
    title: t('notification.yourTurn'),
    timeout: DEFAULT_TOAST_TIMEOUT,
  })
}

const skipTurn = async () => {
  const lastTurnPlayer = players.value.find(p => p.id === state.value.lastTurn)
  const playersWithPenalty = players.value.map(p => {
    if (p.id === lastTurnPlayer?.id) {
      return {
        ...p,
        penalty: Math.max(p.penalty - SKIP_POINT, 0),
      }
    } else if (p.id === user.value.id) {
      return {
        ...p,
        penalty: p.penalty + SKIP_POINT,
      }
    } else {
      return p
    }
  })

  await updateDoc(doc($db, 'matches', matchId), {
    players: playersWithPenalty,
    'state.turn': nextPlayer.value.id,
  })
}

const handleSelectCard = (card: string) => {
  if (isMatchOver.value || !isPlayerTurn.value) {
    return
  }

  if (card === selectedCard.value) {
    selectedCard.value = ''
    turnState.value = TURN_STATE.pickCard
    return
  }

  const canPlaceOnHead = card.includes(head.value)
  const canPlaceOnTail = card.includes(tail.value)

  if (board.value.length !== 0 && canPlaceOnHead && canPlaceOnTail && head.value !== tail.value) {
    selectedCard.value = card
    turnState.value = TURN_STATE.selectPosition
    toast.add({
      title: t('notification.selectPosition'),
      timeout: DEFAULT_TOAST_TIMEOUT,
    })
    return
  }

  if (canPlaceOnHead) {
    putCard(card, BOARD_POSITION.head)
  } else if (canPlaceOnTail) {
    putCard(card, BOARD_POSITION.tail)
  }
}

const handleSelectPosition = (position: BOARD_POSITION) => {
  putCard(selectedCard.value, position)
}

const putCard = async (card: string, position: BOARD_POSITION) => {
  const idx = currentPlayer.value.cards?.indexOf(card)
  currentPlayer.value.cards?.splice(idx, 1)

  if (position === BOARD_POSITION.head) {
    if (card.charAt(0) === head.value) {
      card = card.split('').reverse().join('')
    }
    board.value.unshift(card)
  } else if (position === BOARD_POSITION.tail) {
    if (card.charAt(2) === tail.value) {
      card = card.split('').reverse().join('')
    }
    board.value.push(card)
  }

  const updatedPlayers = players.value.map(p => {
    if (p.id === user.value.id) {
      return currentPlayer.value
    } else {
      return p
    }
  })

  const payload: DocumentData = {
    players: updatedPlayers,
    'state.board': board.value,
    'state.turn': nextPlayer.value.id,
    'state.lastTurn': user.value.id,
  }

  if (currentPlayer.value.cards?.length === 0) {
    const playersWithPenalty = calculatePenalty(match.value, POLDAN_POINT, {
      playerId: currentPlayer.value.id,
      lastCard: card,
    })

    payload.players = playersWithPenalty
    payload['state.turn'] = user.value.id
  } else if (isMatchDraw.value) {
    const playersWithPenalty = calculatePenalty(match.value, GAPLE_POINT)

    payload.players = playersWithPenalty
    payload['state.turn'] = user.value.id
  }

  await updateDoc(doc($db, 'matches', matchId), payload)

  endTurn()
}

const endTurn = () => {
  turnState.value = TURN_STATE.pickCard
  selectedCard.value = ''
}

const newRound = async () => {
  if (!isRoomMaster.value) {
    return
  }

  const shuffledCards = shuffleCards()
  const playersWithCards = players.value.map(player => ({
    ...player,
    cards: shuffledCards.splice(0, 7),
  }))

  const payload: DocumentData = {
    players: playersWithCards,
    'state.board': [],
    'state.round': increment(1),
    'state.turn': state.value.lastTurn,
    'state.lastTurn': '',
    'state.firstTurnCard': '',
  }

  if (rtPlayer.value) {
    payload['state.rt'] = rtPlayer.value.id
    payload.players = payload.players.map(p => ({
      ...p,
      penalty: 0,
    }))
  }

  if (isMatchDraw.value) {
    let lastHead = head.value
    let lastTail = tail.value

    if (lastHead === '0' && lastTail === '0') {
      lastHead = '1'
      lastTail = '1'
    } else if (lastHead === '6' && lastTail === '6') {
      lastHead = '5'
      lastTail = '5'
    }

    const firstTurnCard = [lastHead, lastTail].sort().join('|')
    const firstTurnPlayer = playersWithCards.find(p => p.cards.includes(firstTurnCard))

    payload['state.turn'] = firstTurnPlayer!.id
    payload['state.firstTurnCard'] = firstTurnCard
  }

  await updateDoc(doc($db, 'matches', matchId), payload)
}

const unsubFirestore = () => {
  if (unsubGameplay.value) {
    unsubGameplay.value()
  }
}

onMounted(() => {
  unsubGameplay.value = onSnapshot(doc($db, 'matches', matchId), handleTurn)
})

onBeforeUnmount(() => {
  unsubFirestore()
})
</script>
