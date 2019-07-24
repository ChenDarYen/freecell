<template>
  <div class="main">
    <nav class="sidebar">
      <ul>
        <li>
          <img src="../assets/images/new.svg" width="60px"
          @click="dsiplayModal('newGame')">
        </li>
        <li>
          <img src="../assets/images/restart.svg" width="88px"
          @click="dsiplayModal('restart')">
        </li>
        <li>
          <img src="../assets/images/undo.svg" width="60px"
          @click="undo">
        </li>
      </ul>
    </nav>

    <div class="board"
    :data-action="canAction">
      <div class="cellWall" v-for="index in 16" :key="index">
        <div :class="determineCell(index)"
        :data-name="`${index}-${determineCell(index)}`">
        </div>
      </div>

      <div class="winning"
      v-show="isWin"
      :class="{ visable: isWin }">
        <p>Congratulations</p>

        <div class="btn-container">
          <div>
            <button class="btn"
            @click="newGame">start a new game</button>
          </div>
          <div>
            <button class="btn btn-sub"
            @click="restart">restart this game</button>
          </div>
        </div>
      </div>

      <card-t class="card" opacity="1" borderradius="3%"
      v-for="(card, index) in deck" :key="card.point + card.suit"
      :rank="card.point" :suit="card.suit"
      data-legal data-tableau
      :data-index="index" :data-sibling="JSON.stringify(card.siblings)" :data-cell="card.cell"
      @mousedown="changeZIndex" @mouseup="mouseup"
      v-drag
      ></card-t>

      <div class="logo">
        <img src="../assets/images/logo.svg">
      </div>
    </div>

    <Modal class="modal"
    v-show="showModal"
    :messages="messages" :btnMsg="btnMsg"
    @act="act" @hide="hideModal"/>
  </div>
</template>

<script lang="ts">
import { Component, Vue } from 'vue-property-decorator';
import Modal from '@/components/modal.vue';

class Card {
  public point: number;
  public suit: string;
  public color: string;
  public src: string;
  public index: number;
  public siblings: number[] = [];
  public cell: string = '';

  constructor (point: number, suit: string, i: number) {
    this.point = point;
    this.suit = suit;
    this.index = i;
    this.src = `../assets/images/${point}-${suit}.png`;

    const siblingsMulti: number = Math.floor(this.index / 4) + 1;
    if (suit === 'spades' || suit === 'clubs') {
      this.color = 'black';
      if (this.point < 13) {
        this.siblings = [siblingsMulti * 4 + 1, siblingsMulti * 4 + 2];
      }
    } else {
      this.color = 'red';
      if (this.point < 13) {
        this.siblings = [siblingsMulti * 4 + 0, siblingsMulti * 4 + 3];
      }
    }
  }
}

interface PosConfig {
  x: number;
  y: number;
}
interface TrackConfig {
  [propName: string]: number[];
}
interface ChapterConfig {
  track: TrackConfig;
  from: string;
  emptyCellAmount: number;
  emptyCascadeAmount: number;
}

@Component({
  directives: {
    drag: {
      bind: (element, { value }) => {
        const card: HTMLElement = element;

        card.onmousedown = (e) => {
          e.preventDefault(); // 避免觸發圖片拖曳效果

          const board = document.querySelector('.board') as HTMLElement;
          const canAction: boolean = board.getAttribute('data-action') === 'true';

          if (canAction) {
            const cardIdx = card.getAttribute('data-index');
            const legal: boolean = card.getAttribute('data-legal') === 'true';
            if (legal) {
              e.stopPropagation();

              const cardW: number = card.offsetWidth;
              const cardH: number = card.offsetHeight;
              const boardW: number = document.querySelector('.board')!.clientWidth; // non-null assertion operator !
              const boardH: number = document.querySelector('.board')!.clientHeight;
              const cards: NodeListOf<HTMLElement> = document.querySelectorAll('.card');
              const tableau: number[] = JSON.parse(card.getAttribute('data-tableau') as string);
              const diff: PosConfig[] = [];

              for (const index of tableau) {
                const tableauCard: HTMLElement = cards[index];
                const diffX: number = e.clientX - tableauCard.offsetLeft;
                const diffY: number = e.clientY - tableauCard.offsetTop;
                diff.push({
                  x: diffX,
                  y: diffY,
                });
              }

              document.onmousemove = (event) => {
                let inRange: boolean = true;

                for (const [index] of tableau.entries()) {
                  const { x: diffX }: { x: number } = diff[index];
                  const { y: diffY }: { y: number } = diff[index];
                  if (event.clientX - diffX < 0 || event.clientX - diffX + cardW - boardW > 0
                  || event.clientY - diffY < 0 || event.clientY - diffY + cardH - boardH > 0) {
                    inRange = !inRange;
                    break;
                  }  
                }
                
                if (inRange) {
                  const mouseX: number = event.clientX;
                  const mouseY: number = event.clientY;
                  for (const [index, CardIndex] of tableau.entries()) {
                    const tableauCard: HTMLElement = cards[CardIndex];
                    const { x: diffX }: { x: number } = diff[index];
                    const { y: diffY }: { y: number } = diff[index];
                    tableauCard.style.left = `${mouseX - diffX}px`;
                    tableauCard.style.top = `${mouseY - diffY}px`;  
                  }
                }
              };

              card.onmouseup = () => {
                document.onmousemove = null;
                card.onmouseup = null;
              };
            }  
          }
        };  
      },
    },
  },
  components: {
    Modal,
  },
})

export default class Home extends Vue {
  // data
  private readonly points: number[] = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13];
  private readonly suits: string[] = ['spades', 'hearts', 'diamonds', 'clubs'];
  private deck: Card[] = [];
  private history: ChapterConfig[] = [];
  private cellsTrack: TrackConfig = {};
  private cardDistRatio: number = .2;
  private emptyCellAmount: number = 0;
  private emptyCascadeAmount: number = 0;
  private travelerHomeIndex: string = '';
  private zIndex: number = 53; // 確保比所有牌的 zIndex 都大
  private initializeTime: number = 1;
  private waitingTime: number = .5;
  private canAction: boolean = false;
  private showModal: boolean = false;
  private isWin: boolean = false;
  private action: string = '';
  private messages: string[] = [];
  private btnMsg: string = '';

  constructor () {
    super();
    for (let i = 1 ; i <= 16; i ++) {
      const key = `${i}-${this.determineCell(i)}`;
      this.cellsTrack[key] = [];
    }
  }

  // lifecycle
  public created () {
    this.history = localStorage.getItem('history') ? JSON.parse(localStorage.getItem('history') as string) : [];
    this.createDeck();
  }
  public mounted () {
    // show cards
    const showCards = () => {
      const firstCell = document.querySelector('.cell') as HTMLElement;
      const cards: NodeListOf<HTMLElement> = document.querySelectorAll('.card');
      const cardX: number = firstCell.offsetLeft;
      const cardY: number = firstCell.offsetTop;
      const cardW: number = firstCell.offsetWidth;
      const cardH: number = firstCell.offsetHeight;
      for (const card of cards) {
        card.style.top = `${cardY}px`;
        card.style.left = `${cardX}px`;
        card.style.width = `${cardW}px`;
      }
    };
    showCards();

    if (this.history.length > 0) {
      this.cellsTrack = this.history[this.history.length - 1].track;
      this.dsiplayModal('back');
    } else {
      this.distributeCards();
      setTimeout(() => {
        this.cardsPositioningTableau(1);
      }, this.waitingTime * 1000);
    }

    // 讓卡片隨視窗調整更動大小與位置
    window.onresize = () => {
      showCards();
      this.cardsPositioningTableau(0);
    };
  }

  // methods
  public initialization (immediate: boolean = false) {

    this.isWin = false;
    this.showModal = false;
    this.history = [];
    this.zIndex = 53;
    for (let i = 1 ; i <= 16; i ++) {
      const key = `${i}-${this.determineCell(i)}`;
      this.cellsTrack[key] = [];
    }

    if (!immediate) {
      const firstCell = document.querySelector('.cell') as HTMLElement;
      const cards: NodeListOf<HTMLElement> = document.querySelectorAll('.card');
      const cardX: number = firstCell.offsetLeft;
      const cardY: number = firstCell.offsetTop;

      for (const card of cards) {
        card.style.transition = `all ${this.initializeTime}s`;
        card.style.top = `${cardY}px`;
        card.style.left = `${cardX}px`;
        setTimeout(() => {
          card.style.transition = '';
        }, this.initializeTime * 1000);
      }  
    }
  }
  public determineCell (index) {
    if (index <= 4) {
      return 'cell';
    } else if (index <= 8) {
      return 'foundation';
    } else {
      return 'cascade';
    }
  }
  public createDeck () {
    this.deck = [];
    let i = 0;
    for (const point of this.points) {
      for (const suit of this.suits) {
        this.deck.push(new Card(point, suit, i));
        i = i + 1;
      }
    }
  }
  public recordHistory () {
    const chapter = {
      // 只傳址會導致先前的紀錄被重寫。因為 cellTrack 有兩層， 解構賦值或 Object.asign() 也不行
      track: JSON.parse(JSON.stringify(this.cellsTrack)),
      from: this.travelerHomeIndex,
      emptyCellAmount: this.emptyCellAmount,
      emptyCascadeAmount: this.emptyCascadeAmount,
    };
    this.history.push(chapter);
    localStorage.setItem('history', JSON.stringify(this.history));
  }
  public distributeCards () {
    const cards: NodeListOf<HTMLElement> = document.querySelectorAll('.card');
    const deck = [...this.deck];

    for (let i = 0; i < 52; i ++) {
      const key = `${i % 8 + 9}-cascade`;
      const cardIdx = Math.floor(Math.random() * deck.length);
      this.cellsTrack[key] = [...this.cellsTrack[key], deck[cardIdx].index];
      this.deck[deck[cardIdx].index].cell = key;
      deck.splice(cardIdx, 1);
    }

    this.emptyCellAmount = 4;
    this.emptyCascadeAmount = 0;
    this.recordHistory();
  }
  public writeDataCell (key) {
    const cell: number[] = this.cellsTrack[key];

    for (const cardIndex of cell) {
      this.$set(this.deck[cardIndex], 'cell', key);
    }
  }
  public cardsPositioningTableau (time, ...keys: string[]): void {
    this.canAction = false;

    const positioning = (key) => {
      const selector: string = `[data-name="${key}"]`;
      const cell = document.querySelector(selector) as HTMLElement;
      const cellIndex: number = parseInt(key, 10);
      const cellHeight: number = cell.offsetHeight;
      const cellPosX: number = cell.offsetLeft;
      const cellPosY: number = cell.offsetTop;
      const cellCards: number[] = this.cellsTrack[key];
      const realCards: NodeListOf<HTMLElement> = document.querySelectorAll('.card');

      const moveTo = (cardIndex: number, ratio: number = 0, multiple: number = 0) => {
        const realCard: HTMLElement = realCards[cardIndex];

        realCard.style.transition = `all ${time}s`;
        realCard.style.left = `${cellPosX}px`;
        realCard.style.top = `${cellPosY + multiple * cellHeight * this.cardDistRatio}px`;
        realCard.style.zIndex = `${multiple}`;
        setTimeout(() => {
          realCard.style.transition = '';
        }, time * 1000);
      };

      if (cellCards.length > 0) {
        if (cellIndex < 5) {
          const cardIndex: number = cellCards[0];
          moveTo(cardIndex);
        } else if (cellIndex < 9) {
          for (const cardIndex of cellCards) {
            moveTo(cardIndex);
          }  
        } else {
          for (const [index, cardIndex] of [...cellCards].reverse().entries()) {
            const multiple: number = cellCards.length - index - 1;
            moveTo(cardIndex, this.cardDistRatio, multiple);
          }  
        }  
      }
    };
    const checkTableau = (key) => {
      const cards: NodeListOf<HTMLElement> = document.querySelectorAll('.card');
      const cell: number[] = this.cellsTrack[key];
      let color: string = '';
      let point: number = 0;
      const tableau: number[] = [];

      for (let i = cell.length - 1; i >= 0; i --) {
        const card: Card = this.deck[cell[i]];
        const cardColor: string = card.color;
        const cardPoint: number = card.point;
        const cardIdx: number = cell[i];
        const maxMovingAmount: number = (this.emptyCellAmount + 1) * (this.emptyCascadeAmount + 1);

        if ((i === cell.length - 1 || (cardColor !== color && cardPoint === point + 1))
        && cell.length - i <= maxMovingAmount) {
          tableau.push(cardIdx);
          color = cardColor;
          point = cardPoint;
          cards[cardIdx].setAttribute('data-legal', 'true');
          cards[cardIdx].setAttribute('data-tableau', JSON.stringify(tableau));
        } else {
          cards[cardIdx].setAttribute('data-legal', 'false');
        }
      }
    };

    if (keys.length === 0) {
      for (const [key] of Object.entries(this.cellsTrack)) {
        checkTableau(key);
        positioning(key);
      }
    } else {
      for (const key of keys) {
        checkTableau(key);
        positioning(key);
      }
    }

    setTimeout(() => {
      this.canAction = true;
    }, time * 1000);
  }
  public changeZIndex (e) {
    const isLegal: boolean = e.currentTarget.getAttribute('data-legal') === 'true';

    if (isLegal) {
      const cards: NodeListOf<HTMLElement> = document.querySelectorAll('.card');
      const tableau: number[] = JSON.parse(e.currentTarget.getAttribute('data-tableau') as string);
      const maxZindex: number = this.zIndex + tableau.length;

      for (const [index, cardIndex] of tableau.entries()) {
        cards[cardIndex].style.zIndex = `${maxZindex - index}`;
      }
    }
  }
  public connect (e) {
    const card: HTMLElement = e.currentTarget;
    const isLegal: boolean = card.getAttribute('data-legal') === 'true';

    if (isLegal) {
      const cards: NodeListOf<HTMLElement> = document.querySelectorAll('.card');
      const cardIdx: number = Number(card.getAttribute('data-index'));
      const cardCellKey = card.getAttribute('data-cell') as string;
      const cardPos: PosConfig = { x: card.offsetLeft, y: card.offsetTop };
      const tableau: number[] = JSON.parse(card.getAttribute('data-tableau') as string);
      const tableauAmount: number = JSON.parse(card.getAttribute('data-tableau') as string).length;
      const criterion: number = 50;
      const time: number = .2;
      let isConnect: boolean = false;

      const isClose = (destination: HTMLElement): boolean => {
        const destinationPos: PosConfig = { x: destination.offsetLeft, y: destination.offsetTop };

        if (Math.abs(cardPos.x - destinationPos.x) < criterion && Math.abs(cardPos.y - destinationPos.y) < criterion) {
          return true;
        } else {
          return false;
        }
      };
      const back = () => {
        card.style.zIndex = `${this.zIndex}`;
        this.cardsPositioningTableau(time, cardCellKey);
      };
      const track = (destinationCellKey: string) => {
        const cardCellLeng = this.cellsTrack[cardCellKey].length;
        const travelers: number[] = this.cellsTrack[cardCellKey].splice(cardCellLeng - tableauAmount, tableauAmount);

        this.cellsTrack[destinationCellKey] = [...this.cellsTrack[destinationCellKey], ...travelers];
      };
      const checkEmptyCell = (destinationCellKey) => {
        const cardCellIdx: number = parseInt(cardCellKey, 10);
        const destinationcellIdx: number = parseInt(destinationCellKey, 10);

        if (this.cellsTrack[cardCellKey].length === 0) {
          if (cardCellIdx < 5) {
            this.emptyCellAmount = this.emptyCellAmount + 1;
          } else if (cardCellIdx > 8) {
            this.emptyCascadeAmount = this.emptyCascadeAmount + 1;
          }
        }
        if (this.cellsTrack[destinationCellKey].length === tableauAmount) {
          if (destinationcellIdx < 5) {
            this.emptyCellAmount = this.emptyCellAmount - 1;
          } else if (destinationcellIdx > 8) {
            this.emptyCascadeAmount = this.emptyCascadeAmount - 1;
          }
        }
      };
      const travel = (travelTime: number, destinationCellKey: string, ratio: number = 0) => {
        track(destinationCellKey);
        checkEmptyCell(destinationCellKey);
        this.travelerHomeIndex = cardCellKey;
        this.cardsPositioningTableau(travelTime);
        this.writeDataCell(destinationCellKey);
        this.recordHistory();

        isConnect = true;
      };

      // check siblings
      const siblings: number[] = JSON.parse(card.getAttribute('data-sibling') as string);
      
      for (const siblingIdx of siblings) {
        const sibling = cards[siblingIdx] as HTMLElement;
        const siblingCard: Card = this.deck[sibling.getAttribute('data-index') as string];
        const siblingCellIdx = sibling.getAttribute('data-cell') as string;
        const siblingCell: number[] = this.cellsTrack[siblingCellIdx];
        const isCover: boolean = siblingCell[siblingCell.length - 1] === siblingCard.index;

        if (isCover && isClose(sibling)) {
          travel(time, siblingCellIdx, this.cardDistRatio);
          break;
        }
      }

      // check cell
      if (!isConnect) {
        for (const [key, cellData] of Object.entries(this.cellsTrack)) {
          const cell = document.querySelector(`[data-name="${key}"]`) as HTMLElement;
          const cellLeng: number = cellData.length;
          const index: number = parseInt(key, 10);
          if (index < 5) { // cells
            if (cellLeng === 0 && tableauAmount === 1 && isClose(cell)) {
              travel(time, key);
              break;
            }
          } else if (index < 9) { // foundations
            const CoverCardIdx: number = cellData.length ? cellData[cellData.length - 1] : -10;

            if ((cardIdx < 5 || cardIdx === CoverCardIdx + 4) && isClose(cell)) {
              travel(time, key);
              break;
            }
          } else { // cascades
            if (cellLeng === 0 && tableauAmount <= this.emptyCellAmount + 1 && isClose(cell)) {
              travel(time, key);
              break;
            }
          }
        }

        // if card cannot connect, back!
        if (!isConnect) {
          back();
        } else {
          // check thie status of the game
          this.checkStatus();
        }
      }
    }
  }
  public checkStatus () {
    let isWin: boolean = true;

    for (let i = 0; i < 4; i ++) { // 檢查四張 k 是否都在 foundation 裡
      const cardIndex: number = 51 - i;
      const cardCellIdx: number = parseInt(this.deck[cardIndex].cell, 10);

      if (cardCellIdx < 5 || cardCellIdx > 8) {
        isWin = false;
        break;
      }
    }

    if (isWin) {
      this.isWin = true;

      this.$nextTick(() => { // 確保 winning 的長寬已被設置
        const winning = document.querySelector('.winning') as HTMLElement;
        const winningW: number = winning.offsetWidth;
        const boardW: number = document.querySelector('.board')!.clientWidth;
        const foundationY: number = (document.querySelector('.foundation') as HTMLElement).offsetTop;
        const cardH: number = (document.querySelector('.card img') as HTMLElement).offsetHeight;

        winning.style.left = `${(boardW - winningW) / 2}px`;
        winning.style.top = `${foundationY + cardH + 60}px`;
      });
    }
  }
  public newGame (immediate: boolean = false) {
    const time: number = immediate ? 0 : this.initializeTime + this.waitingTime;

    if (this.canAction) {
      this.canAction = false;
      this.initialization(immediate);
      setTimeout(() => {
        this.createDeck();
        this.distributeCards();
        this.cardsPositioningTableau(1); 
        this.canAction = true;     
      }, time * 1000);  
    } 
  }
  public restart () {

    if (this.canAction) {
      this.canAction = false;
      
      const firstTrack: TrackConfig = this.history[0].track;
      this.initialization();
      setTimeout(() => {
        this.createDeck();
        this.cellsTrack = firstTrack;
        this.cardsPositioningTableau(1); 
        this.recordHistory();
        this.canAction = true;     
      }, (this.initializeTime + this.waitingTime) * 1000);  
    }
  }
  public undo () {
    if (this.history.length > 1) { // 為保留最初的資料設置條件 > 1
      const chapter: ChapterConfig = this.history[this.history.length - 2];
      this.cellsTrack = JSON.parse(JSON.stringify(chapter.track));
      this.emptyCellAmount = chapter.emptyCellAmount;
      this.emptyCascadeAmount = chapter.emptyCascadeAmount;
      this.writeDataCell(this.history[this.history.length - 1].from); // 需在刪除資料前執行
      this.history.pop();
      this.cardsPositioningTableau(.5);
      localStorage.setItem('history', JSON.stringify(this.history));
    }
  }
  public back () {
    this.canAction = true;
    this.newGame(true);
  }
  public hideModal () {
    this.showModal = false;
    if (this.action === 'back') {
      this.emptyCellAmount = this.history[this.history.length - 1].emptyCellAmount;
      this.emptyCascadeAmount = this.history[this.history.length - 1].emptyCascadeAmount;
      this.cardsPositioningTableau(1);
      for (let i = 9; i <= 16; i ++) {
        const key: string = `${i}-cascade`;
        this.writeDataCell(key);
      }
    }
  }
  public act () {
    this[this.action]();
  }
  public dsiplayModal (action: string) {
    this.action = action;
    if (action === 'newGame') {
      this.messages = ['Do you want to start a new game?', 'This will quit the current game.'];
      this.btnMsg = 'start a new game';
      this.showModal = true;
    } else if (action === 'restart') {
      this.messages = ['Do you want to restart the current game?', 'This will undo all your moves.'];
      this.btnMsg = 'restart the current game';
      this.showModal = true;
    } else if (action === 'back') {
      this.messages = ['You have a game in progress.'];
      this.btnMsg = 'start a new game';
      this.showModal = true;
    }
    (document.querySelector('.modal') as HTMLElement).style.zIndex = `${this.zIndex}`;
  }
  public mouseup (e) {
    this.connect(e);
    this.checkStatus();
  }
}
</script>

<style lang="scss" scoped>
@import '../assets/all';

.main {
  width: 100%;
  height: 100%;
  background: linear-gradient(#525252, #001F1D);
  position: relative;
  display: inline-flex;
  flex-direction: row-reverse;
}
.sidebar {
  width: $sidebarWidth;
  height: 100%;
  padding-top: $spacing * 2;
  background-color: #001F1D;
  text-align: center;
  img {
    margin-bottom: 50px;
  }
}
.board {
  width: calc(100% - #{$sidebarWidth});
  height: 100%;
  padding: ($logoHeight + $spacing * .5) ($spacing * 2) $spacing;
  display: inline-flex;
  flex-wrap: wrap;
  align-items: flex-start;
  align-content: flex-start;
  position: relative;
}
.logo {
  height: $logoHeight;
  margin-left: $spacing;
  position: absolute;
  top: 0;
  left: 0;
  img {
    height: 100%;
  }
}
.cellWall {
  width: $cardWidth;
  margin-right: calc((100% - 8 * #{$cardWidth}) / 7);
  margin-bottom: 15px;
  opacity: .5;
  &:nth-child(8n) {
    margin-right: 0;
  }
}
.cell, .foundation, .cascade {
  width: 100%;
  padding-bottom: $cardHeightRatio;
  border-radius: $cardBorderRadius;
  border: 1px solid #ffffff;
}
.cell, .cascade {
  position: relative;
  &::after {
    width: 95%;
    padding-bottom: $cardHeightRatio * .95;
    display: block;
    content: "";
    border-radius: $cardBorderRadius;
    border: 1px solid #ffffff;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }
}
.foundation{
  position: relative;
  &::after {
    display: block;
    content: "A";
    color: #ffffff;
    font-weight: 900;
    font-size: 5vw;
    position: absolute;
    top: 50%;
    left: 50%;
    transform: translate(-50%, -50%);
  }  
}
.card {
  width: 0;
  height: 0;
  position: absolute;
}
.modal {
  position: absolute;
  top: 0;
  left: 0;
}
.winning {
  opacity: 0;
  text-align: center;
  position: absolute;
  transition: all 1s;
  p {
    font-size: 2rem;
    color: #ffffff;
    text-transform: capitalize;
  }
  .btn-container {
    margin-top: 60px;
  }
}
.visable {
  opacity: 1;
}
</style>