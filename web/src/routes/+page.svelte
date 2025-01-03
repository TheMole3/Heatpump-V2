<script>
  // Trigger page initialization when the component is mounted
  import { onMount } from "svelte";
  import { fetchUserInfo } from "$lib/utils.js"; // Assuming you have a function getUserInfo to fetch user info based on the token
  import { getUpdatedToken, logout, startOidcFlow } from "./AuthManager.js";
  import HeatPumpControl from "./HeatPumpControl.svelte";
  import Fa from 'svelte-fa'
  import { faSignOut } from '@fortawesome/free-solid-svg-icons'
  import {initializeMQTT, mqttError} from './MqttManager.js'
  import Graph from './Graph.svelte'

  let userInfo = null;
  let loading = true; // Manage loading state locally
  let loginError = false; // Manage loginError state locally

  // Function to initialize the page and check for valid token
  async function initializePage() {
    try {
      const {
        accessToken,
        loading: tokenLoading,
        loginError: tokenLoginError,
      } = await getUpdatedToken();

      if (tokenLoading) {
        loading = true;
        return; // Wait until loading is complete
      }

      if (tokenLoginError) {
        loginError = tokenLoginError;
        loading = false;
        return; // Handle login error
      }

      if (accessToken) {
        userInfo = await fetchUserInfo(accessToken); // Get user info with the valid token
        await initializeMQTT();
      } else {
        throw new Error("No valid access token.");
      }
    } catch (err) {
      console.error("Error initializing page:", err);
      loginError = err;
      loading = false;
    } finally {
      loading = false;
    }
  }

  onMount(() => {
    initializePage();
  });

  // Function to handle login redirect if the token is invalid
  function handleLoginRedirect() {
    startOidcFlow();
  }
</script>

<div class="w-full min-h-screen bg-base-200 select-none flex justify-center">
  <div class="w-screen md:w-7/12 lg:w-5/12 sm flex flex-col items-center">
    {#if loading}
      <div class="loading mt-24">Loading...</div>
    {:else if loginError}
      <h1 class="mt-10">Login failed. Please try again.</h1>
      <button class="btn btn-secondary rounded-sm mt-5" on:click={handleLoginRedirect}>Try Logging In Again</button>
      <p class="mt-10 text-red-500">{loginError}</p>
    {:else if $mqttError}
      <h1 class="mt-10">MQTT connection failed.</h1>
      <button class="btn btn-secondary rounded-sm mt-5" on:click={handleLoginRedirect}>Try Logging In Again</button>
      <p class="mt-10 text-red-500">{$mqttError}</p>
    {:else if userInfo}
      <button class="flex items-center gap-3 mt-2" on:click={() => logout()}
        >Hej {userInfo.name.split(" ")[0]}! Klicka här för att logga ut
        <Fa class="mt-1" icon={faSignOut}/>
      </button>
      <HeatPumpControl></HeatPumpControl>
      <div class="h-96 w-8/12">
        <Graph></Graph>
      </div>
    {/if}
  </div>
</div>
