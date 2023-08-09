<script lang="ts">
  export let onClickEvent;
  export let currentValue = 0;
  let displayValue = "";

  $: {
    console.log(currentValue);
    if (currentValue === 1) displayValue = "X";
    else if (currentValue === 2) displayValue = "O";
    else displayValue = "";
  }
</script>

<button on:click={onClickEvent}>{displayValue}</button>
