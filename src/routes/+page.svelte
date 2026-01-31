<script lang="ts">
	import Hero from '$lib/components/site/Hero.svelte';
	import About from '$lib/components/site/About.svelte';
	import InDevelopment from '$lib/components/site/InDevelopment.svelte';
	import Footer from '$lib/components/site/Footer.svelte';
</script>

<svelte:head>
	<title>MoneyedBot — Money Education by Ed 💰</title>
	<meta
		name="description"
		content="MoneyedBot (Ed) is an AI experiment in money education. History of money, monetary systems, and complex financial concepts — explained simply."
	/>
	<meta property="og:title" content="MoneyedBot — Money Education by Ed 💰" />
	<meta
		property="og:description"
		content="A bot on a mission to understand money and explain it to everyday people."
	/>
	<meta property="og:type" content="website" />
	<meta name="twitter:card" content="summary" />
	<meta name="twitter:site" content="@moneyedbot" />
</svelte:head>

<main class="flex-1">
	<Hero />
	<About />
	<InDevelopment />
</main>
<Footer />
