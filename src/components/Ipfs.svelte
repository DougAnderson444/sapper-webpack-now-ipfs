<script>
  var Buffer = require('buffer/').Buffer // note: the trailing slash is important!

  // svelte stuff
  import { onMount } from 'svelte'

  //stores
  import {
    nodeId,
    nodeAgentVersion,
    nodeProtocolVersion,
    ipfsNode,
    rootHash,
  } from './stores.js'

  // IPFS
  import IPFS from 'ipfs'
  const IpfsHttpClient = require('ipfs-http-client')

  let password = ''
  let modifier = ''
  export let addedFileContents, ipfsAPI
  export let ipfsApiId = 'Loading...'

  const nodeConnect = async (apiMultiAddr) => {
    console.log('IpfsHttpClient(apiMultiAddr)', apiMultiAddr)
    try {
      ipfsAPI = IpfsHttpClient(apiMultiAddr)
      const { id } = await ipfsAPI.id()
      ipfsApiId = `Remote Node ID: ` + id
      console.log(`ipfsAPI`, ipfsApiId)
    } catch (error) {
      console.log(error)
    }
  }

  onMount(async () => {
    // from ipfs browser-webpack
    // name the repository, repository saved in the browser's IndexedDB
    // https://github.com/ipfs/js-ipfs/tree/master/packages/ipfs#ipfs-constructor

    const options = {
      repo: 'ipfs' + modifier, // default is "ipfs", string or ipfs.Repo instance, file path at which to store the IPFS node’s data, String(Math.random() + Date.now())
      pass: password, // https://github.com/ipfs/js-ipfs/issues/1138
      //init: {				// only runs initially
      //privateKey: privKey, // (base64 PrivKey) string or full PeerId, A pre-generated private key to use. Can be either a base64 string or a PeerId instance.
      //}
    }

    /*
     * START IPFS NODE
     */
    $ipfsNode = await IPFS.create(options)
    console.log(`ipfs node ready \n`, $ipfsNode)

    console.log(`API to `, process.env.SAPPER_APP_API_URL)
    await nodeConnect(process.env.SAPPER_APP_API_URL)

    const { id, agentVersion, protocolVersion } = await $ipfsNode.id()

    //copy to svelte stores
    $nodeId = id

    const result = await $ipfsNode.add('Saved some data to IPFS, cool!')
    console.log(result.cid)
    $rootHash = result.cid.toString()

    const stream = $ipfsNode.cat(result.cid)
    let data = ''

    for await (const chunk of stream) {
      // chunks of data are returned as a Buffer, convert it back to a string
      data += chunk.toString()
    }

    console.log(data)
    addedFileContents = data.toString('utf8')
  })
</script>

<style>
  div.outer {
    outline: 1px solid lightgray;
    padding: 15px;
  }
</style>

<div class="outer">
  {#if $nodeId}
    <div>
      <h2>Your node is running in the browser.</h2>
      <p>
        {#if $rootHash}
          <b>
            View your data in web 3.0'land (kinda like a blockchain, but
            different):
          </b>
          <br />
          <a
            target="_blank"
            rel="noopener noreferrer"
            href="https://explore.ipld.io/#/explore/{$rootHash}">
            {$rootHash}
          </a>
        {/if}
      </p>
      <p>Browser nodeID: {$nodeId}</p>
      <p>{addedFileContents}</p>
      <h2>Let's connect to a remote IPFS server, too!</h2>
      <p>{ipfsApiId}</p>
    </div>
  {:else}
    <div>
      <h2>Loading your peer node...</h2>
    </div>
  {/if}
</div>
