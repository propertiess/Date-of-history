<script>
  import * as jsonDates from "../public/data/refactored_dates.json";
  import { onMount } from 'svelte';
  import {createSmartappDebugger, createAssistant} from '@sberdevices/assistant-client'
  const dates = jsonDates.dates

  let level = 9;
  let levelDates = [];
  let currentQuestion = {
    answers: [],
    rightAnswer: 0
  };
  let questionNumber = 1;
  let lives = 3;
  let lost = false;
  let won = false;
  let wait = false;

  let lastRightAnswer = 0;
  let gameStarted = false;
  let assistant;
  let token = 'eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiJmZjAwMTQyZDBiMDYxMDEzNjk1NGNkYmMzM2E5MzQ4YjA2NzAxMTRhZTQ4MGIzYWIyMWUzOGU1MWMzYzRmZDEyMjFhZGQiLCJhdWQiOiJWUFMiLCJleHAiOjE2NDE0MTA1ODQsImlhdCI6MTY0MTMyNDE3NCwiaXNzIjoiS0VZTUFTVEVSIiwidHlwZSI6IkJlYXJlciIsImp0aSI6IjhlMGU0NDdjLWU2NzktNGEyMC1hOWVmLTU0NzUxMzQ0MmUyMyIsInNpZCI6ImFhMGIwZDRiLTIxYzAtNGNkOC1iZDJkLTRjZGU2YTQyOTAxMiJ9.Gie6fQyWD1aDOLv0hL5-heUkkF5rT83Vgr5FiGEIk2TtGX0atskCFXFk_qqM-hclAfR-wH74lJd5_MeHKthOkGKWiFmUkmGFs6A_27RKe_9rJfxJtAicSbjjqbmzxS3ansVs2-nSvhi4TEptoq4WnFpzsZqwm4koRkyd2n06eD-Roswt33u3n8vb8Wnz0uzadNDJsc5MakngSE6GUceF26FO2DYy-fjrM7TZqu8-W_AUo59CUHMgPONWVnwZQ3-LcHli085Tca2AbibCtTLKuNywKAXvdCBZU7bqxnpM2HzZsoW3FMWynJq25c5qSbrOeXfFsNgnM7Fw9t8sFV0HP2E9UTlba5H7dqRuQ95OzVzcWHnxKC-5PhJ92FHBJquIy3hFHpV4b_31BRX2EQOXgQeVpaofRaUvLSPyiLvUfybvRmCyr9-APbunEEZTVLsqdBoDdQ_wxK5yWP89MXm_ULG6GIp2pwwqRAym1cdhytyMhFNBhX21FHdGkC-hLo0jh07U3Xr2P9pCMU1oss_p6cZuDSEpScdF2WcQ5btZC7g_fDvJsm6bVwDWc1ixyktAf5qaXPSYgX_CbZoKdrdijU8rD8DrkJpDR3nyHHyWaoiaYuxikU8dLNIO6u5CTUVqF0BPPWTTaT1UAq6oeuHWkFMWykrQ70xlSQShym7sV4o';
  let initPhrase = 'Включи Даты в истории'

  function getState() {
    console.log("State was get");
    const state = {
      item_selector: {
        items: [
          currentQuestion
        ]
      }
    }
    return state;
  }


  onMount(() => {
    const init = () => {
      // return createSmartappDebugger({
      //   token,
      //   initPhrase,
      //   getState,
      //   settings: {}
      // })
      return createAssistant({getState});
    }
    assistant = init();
    assistant.on("start", (event) => {
      console.log(`assistant.on(start)`, event);
    });
    assistant.on("data", (event) => {
      console.log('EVENT!!!', event);
      if (!event.action)
        return;
      switch (event.action.type) {
        case 'answer':
          checkAnswer(event.action.answer)
          break;
        case 'restart':
          restart();
      }
    })

    getDates();
  })

  function shuffle(array) {
    return array.sort(() => 0.5 - Math.random());
  }

  function getDates() {
    let shuffled = shuffle(dates[level+''])
    shuffled = shuffled.filter((date) => {return isNaN(date.event[1])})
    levelDates = shuffled.slice(0, 10);
  }

  function createQuestion() {
    const questionDate = levelDates[questionNumber-1];
    const rightAnswer = parseInt(questionDate.date)
    lastRightAnswer = rightAnswer
    let answers = [rightAnswer]

    while(answers.length < 4) {
      const lowerBound = (level-1) * 100;
      let currentBound = rightAnswer % lowerBound;
      if (currentBound < 10)
        currentBound = 100 - currentBound;
      let randomDate = Math.floor(Math.random() * currentBound);
      if (currentBound < 90 && currentBound + randomDate > 100)
        randomDate *= -1;
      if (answers.indexOf(randomDate + rightAnswer) == -1 && !(level === 21 && randomDate + rightAnswer > 2021))
        answers.push(rightAnswer + randomDate);
    }
    // Костыль, чтобы убрать точки в конце
    let question = questionDate.event
    while(true) {
      if (question[question.length-1] === '.') {
        console.log(question)
        question = question.slice(0, question.length-1);
        console.log(question)
      }
      else
        break;
    }

    answers = shuffle(answers);
    currentQuestion = {
      question,
      rightAnswer,
      lastRightAnswer,
      answers
    }

    currentQuestion = currentQuestion;

    wait = false;
    assistant.sendData({action: {action_id: 'voiceQuestion'}});
  }

  function checkAnswer(answer) {
    if (wait)
      return;
    wait = true;
    let timeout = 2000;
    if (currentQuestion.rightAnswer === answer) {
      assistant.sendData({action: {action_id: 'right'}});
    } else {
      assistant.sendData({action: {action_id: 'wrong'}});
      timeout = 5500;
      if (lives === 1) {
        setTimeout(() => lost = true, timeout);
        return;
      }
      lives--;
    }
    if (questionNumber === 10) {
      setTimeout(newLevel, timeout)
      return;
    }
    setTimeout(() => {questionNumber++; createQuestion();}, timeout)
  }

  function newLevel() {
    if (level === 21) {
      won = true;
      return;
    }
    lives += 3;
    level++;
    questionNumber = 1;
    getDates();
    createQuestion();
  }

  function restart() {
    lives = 3;
    level = 9;
    questionNumber = 1;
    getDates();
    createQuestion();
    lost = false;
    won = false;
    gameStarted = true;
  }
</script>

<main>
  {#if gameStarted}
    <div class="component">
      <div id="game-grid">
        <div id="level">
          <h1>{level} век</h1>
        </div>
        <div id="questions-number">
          <h1>{questionNumber}/{levelDates.length}</h1>
        </div>
        <div id="lives-block">
          <div>
            <img src="/heart.png" alt="Жизней: ">
            <h1>{lives}</h1>
          </div>
        </div>
        <div id="questions">
          <h3>{currentQuestion.question}</h3>
          {#each currentQuestion.answers as answer}
            <button class="question sliding-button" on:click={() => checkAnswer(answer)}>
              {answer}
            </button>
          {/each}
        </div>
      </div>
      {#if lost}
        <div class="masc">
          <div class="modal">
            <h1>Проигрыш! Получилось дожить до {level} века. Еще попытку?</h1>
            <button class="sliding-button" on:click={restart}>Начать заново</button>
          </div>
        </div>
      {/if}

      {#if won}
        <div class="masc">
          <div class="modal">
            <h1>Победа! Осталось жизней: {lives}</h1>
            <button class="sliding-button" on:click={restart}>Начать заново</button>
          </div>
        </div>
      {/if}
    </div>
  {:else}
    <div id="start-block">
      <h1>Исторические даты</h1>
      <button class="sliding-button" on:click={() => {gameStarted = true; createQuestion();}}>Начать игру</button>
    </div>
  {/if}
</main>

<style>
	main {
    position: absolute;
    top: 0;
		width: 100%;
    min-height: 100%;
    background-color: #333333;
	}

  #start-block {
    text-align: center;
    margin: 40vh auto 0 auto;
  }

	@media (min-width: 640px) {
		main {
			max-width: none;
		}
	}

  #game-grid {
    width: 100%;
    min-height: 100%;
    display: grid;
    grid-template-areas:
        "questionNumber level lives"
        "questions questions questions";

    grid-template-rows: 1fr 10fr;
    grid-template-columns: 1fr 1fr 1fr;
  }

  #level {
    grid-area: level;
    margin-top: 3vh;
  }

  #questions-number {
    grid-area: questionNumber;
    display: flex;
    justify-content: left;
    margin-left: 50px;
    margin-top: 3vh;
  }

  #lives-block {
    display: flex;
    justify-content: right;
    align-items: center;
    grid-area: lives;
    margin-right: 50px;
    margin-top: 3vh;
  }

    #lives-block div {
      display: flex;
      align-items: center;
      height: 7vw;
    }

  #lives-block img {
    max-height: 100%;
  }

  #lives-block h1 {
    font-size: 7vw;
  }

  #questions {
    grid-area: questions;
    display: flex;
    flex-direction: column;
    align-items: center;
    color: #FFFFFF;
    font-family: Arial;
    margin: 50px auto;
    text-align: center;
    max-width: 70%;
  }

  #questions button {
    width: 100%;
  }

  .question {
    margin: 1vw 0;
  }

  .masc {
    width: 100%;
    height: 100%;
    position: absolute;
    top: 0;
    left: 0;
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 100;
    background-color: #333333;
  }

  .modal {
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
  }

  .modal h1 {
    margin-bottom: 1vw;
  }

  @media (max-width: 1000px) {
    #questions {
      font-size: 6vw;
    }
    #questions h3{
      font-size: 5.5vw;
    }
  }

  @media (max-width: 800px) {
    #lives-block img {
      height: 4.5vh;
    }
  }

  @media (max-width: 400px) {
    #questions {
      font-size: 6.3vw;
    }
  }
</style>