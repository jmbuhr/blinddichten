<script>
  export let wsHandler;
  export let publicPlayerId;
  export let gameId;

  import debounce from '../util/debounce';
  import { onDestroy, createEventDispatcher } from 'svelte';
  import PlayerList from '../controls/PlayerList.svelte';
  import ReadyButton from '../controls/ReadyButton.svelte';
  import { fly } from 'svelte/transition';
  import { addNewGame } from '../util/lastGames';

  const dispatch = createEventDispatcher();

  let players = [];
  let rounds;
  let owner;
  let ready = false;
  let name;

  function lobbyUpdated(msg) {
    players = msg.players;
    rounds = msg.settings.rounds;
    const me = players.find(p => p.publicPlayerId === publicPlayerId);
    ready = me.ready;
    owner = me.isOwner;
    name = me.name;
  }
  wsHandler.on('lobbyUpdated', lobbyUpdated);

  function lobbyCompleted() {
    addNewGame(gameId, players);
  }
  wsHandler.on('lobbyCompleted', lobbyCompleted);

  onDestroy(() => {
    wsHandler.off('lobbyUpdated', lobbyUpdated);
    wsHandler.off('lobbyCompleted', lobbyCompleted);
  });

  function editName(event) {
    name = event.detail;
    wsHandler.sendMessage({
      type: 'changeLobby',
      name: event.detail
    });
  }

  function editRounds(event) {
    wsHandler.sendMessage({
      type: 'changeLobby',
      settings: { rounds: event.target.value }
    });
  }
  const editRoundsDebounced = debounce(editRounds, 1000);

  function readyLobby() {
    if (name) {
      window.localStorage.setItem('playerName', name);
      wsHandler.sendMessage({ type: 'readyLobby' });
    } else {
      dispatch('errorOccurred', 'nothingEntered');
    }
  }
  function unreadyLobby() {
    wsHandler.sendMessage({ type: 'unreadyLobby' });
  }
</script>

<main in:fly="{{x: 500, delay: 400}}" out:fly="{{x: -500}}">
  <h1>Lobby</h1>
  <settings class="card">
    <div class="card-body">
      <h4>Einstellungen</h4>
      <div class="form-group">
        <label for="roundsInput">Rundenzahl</label>
        <input class="input-block" id="roundsInput" disabled="{!owner || ready}" type="number" value="{rounds}" on:keyup="{editRoundsDebounced}" on:change="{editRounds}" />
      </div>
    </div>
  </settings>
  <PlayerList players="{players}" publicPlayerId="{publicPlayerId}" on:editName="{editName}" />
  <ReadyButton on:lock="{readyLobby}" on:unlock="{unreadyLobby}" players="{players}" publicPlayerId="{publicPlayerId}" lastReadyText="Spiel starten" />
</main>

<style>
  main {
    max-width: 1000px;
  }
  @media(min-width: 800px) {
    settings {
      width: 20rem;
      float: right;
    }
  }
</style>
