# freecell

簡介：
---
嘗試使用 Vue-cli + TypeScript 撰寫的新接龍小遊戲。

DEMO
---
https://chendaryen.github.io/freecell/#/

功能：
---
卡片拖曳：
```TS
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
              
              // data-tableau 記錄所有要一起移動的牌的索引
              const tableau: number[] = JSON.parse(card.getAttribute('data-tableau') as string);
              const diff: PosConfig[] = []; // 用來記錄所有要移動的牌和鼠標的差距
              // ...
              document.onmousemove = (event) => {
                // ...判斷是否所有 tableau 內的牌都在範圍內
                // ...如果是，移動所有 tableau 內的牌
              }
              card.onmouseup = () => {
                document.onmousemove = null;
                card.onmouseup = null;
              };
            }
          };
        }
      },
    },
  },
})
```
卡片結合：
```TS
public connect (e, isRemote: boolean = false) {
  // ...
}
```
提示：
```TS
public hint () {
  if (this.hints.hints.length === 0) {
    this.findHints();
    this.showHint();
  } else {
    this.showHint();
  }
}
```
點擊卡片自動移位：
```JS
public mousedown (e) {
  const currentCard: HTMLElement = e.currentTarget;

  this.currentCardPos = {
    x: currentCard.offsetLeft,
    y: currentCard.offsetTop,
  };
  this.changeZIndex(e);
}
public mouseup (e) {
  const currentCard = e.currentTarget;
  
  // 卡片位置沒有移動時執行遠程連接，如果有移動需判斷離可連接目標是否夠近。
  if (currentCard.offsetLeft === this.currentCardPos.x && currentCard.offsetTop === this.currentCardPos.y) {
    this.connect(e, true);
  } else {
    this.connect(e, false);
  }

  this.checkStatus(); // 判斷是否勝利
}
```
上一步：
```TS
public undo () {
  if (this.history.length > 1) { // 為保留最初的資料設置條件 > 1
    const chapter: ChapterConfig = this.history[this.history.length - 2]; // 取得上一步的資料

    this.cellsTrack = JSON.parse(JSON.stringify(chapter.track));
    this.emptyCellAmount = chapter.emptyCellAmount;
    this.emptyCascadeAmount = chapter.emptyCascadeAmount;
    this.writeDataCell(this.history[this.history.length - 1].from); // 需在刪除資料前執行
    this.history.pop();
    this.cardsPositioningTableau(.5);
    localStorage.setItem('history', JSON.stringify({
      timer: this.timer,
      history: this.history,
    }));
    this.resetHints(); // 清空提示
  }
}
```
計時：
```TS
private timer: TimerConfig = {
  display: '00:00',
  count: 0,
  timer: 0,
}

public startTimer () {
  this.$set(this.timer, 'timer', setInterval(() => {
    const count: number = this.timer.count + 1;
    const sec: string = count % 60 > 9 ? `${count % 60}` : `0${count % 60}`;
    const min: string = Math.floor(count / 60) > 9 ? `${Math.floor(count / 60)}` : `0${Math.floor(count / 60)}`;

    this.$set(this.timer, 'count', count);
    this.$set(this.timer, 'display', `${min}:${sec}`);
    this.recordHistory(false);
  }, 1000));
}
public stopTimer () {
  clearInterval(this.timer.timer);
}
```
