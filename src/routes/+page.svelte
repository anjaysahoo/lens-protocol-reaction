<script>

    import { ethers } from 'ethers';
    import {createClient} from "@urql/svelte";

    const API_URL = 'https://api.lens.dev'
    const challenge =`
    query Challenge($address: EthereumAddress!) {
        challenge(request: { address: $address }) {
            text
        }
    }
`
    const authenticate =`
    mutation Authenticate(
        $address: EthereumAddress!
        $signature: Signature!
    ) {
        authenticate(request: {
            address: $address,
            signature: $signature
        }) {
            accessToken
            refreshToken
        }
    }
`

    let address;
    let accessTokenFromLens;
    let isConnected = false;
    let signer;

    let client = createClient({
        url: API_URL
    });

    /**
     * 1. Connect Wallet
     */
    async function connect() {
        console.log("connect called")
        /* this allows the user to connect their wallet */
        try {
            const account = await window.ethereum.send('eth_requestAccounts')
            if (account.result.length) {
                address = account.result[0];
                isConnected = true;
            }
            else{
                isConnected = false;
            }
        }
        catch (error) {
            console.log(error)
        }

    }
    /********************************************************/


    /**
     * 2. Get Access Token
     */
    async function getAccessTokenFromLens() {
        try {
            /* first request the challenge from the API server */
            const challengeInfo = await client.query(challenge, { address }).toPromise();
            const provider = new ethers.providers.Web3Provider(window.ethereum);
            signer = provider.getSigner()
            /* ask the user to sign a message with the challenge info returned from the server */
            const signature = await signer.signMessage(challengeInfo.data.challenge.text);
            /* authenticate the user */
            const authData = await client.mutation(authenticate, { address, signature }).toPromise();
            /* if user authentication is successful, you will receive an accessToken and refreshToken */
            const { data: { authenticate: { accessToken }}} = authData
            console.log({ accessToken })
            accessTokenFromLens = accessToken;

            /** you can now use the accessToken to make authenticated requests to the API server **/
            /** Update client with new accessToken **/
            client = createClient({
                url: API_URL,
                fetchOptions: {
                    headers: {
                        'x-access-token': `Bearer ${accessTokenFromLens}`
                    },
                },
            });
        } catch (err) {
            console.log('Error signing in: ', err)
        }
    }
    /********************************************************/

    /**
     * 3. Get Profile
     */

    const getDefaultProfile = `
query DefaultProfile($address: EthereumAddress!) {
  defaultProfile(request: { ethereumAddress: $address}) {
    id
    name
    bio
    isDefault
    attributes {
      displayType
      traitType
      key
      value
    }
    followNftAddress
    metadata
    handle
    picture {
      ... on NftImage {
        contractAddress
        tokenId
        uri
        chainId
        verified
      }
      ... on MediaSet {
        original {
          url
          mimeType
        }
      }
    }
    coverPicture {
      ... on NftImage {
        contractAddress
        tokenId
        uri
        chainId
        verified
      }
      ... on MediaSet {
        original {
          url
          mimeType
        }
      }
    }
    ownedBy
    dispatcher {
      address
      canUseRelay
    }
    stats {
      totalFollowers
      totalFollowing
      totalPosts
      totalComments
      totalMirrors
      totalPublications
      totalCollects
    }
    followModule {
      ... on FeeFollowModuleSettings {
        type
        contractAddress
        amount {
          asset {
            name
            symbol
            decimals
            address
          }
          value
        }
        recipient
      }
      ... on ProfileFollowModuleSettings {
       type
      }
      ... on RevertFollowModuleSettings {
       type
      }
    }
  }
}

`
    let profile;

    const getUserProfile = async () => {
        try {
            console.log("Get User Profile Called")
            const response = await client.query(getDefaultProfile, {
                address
            }).toPromise()
            profile = response.data.defaultProfile;
            console.log("profile : " + JSON.stringify(profile));
        } catch (err) {
            console.log('error fetching user profile...: ', err)
        }
    }

    /*********************************************************/

    const addReaction = `
    mutation AddReaction($request: ReactionRequest!) {
        addReaction(request: $request)
    }
`

    const addReactionRequest = async (request) => {

        let result = await client.mutation(addReaction, {
            request
        }).toPromise();

        result = result.data.addReaction;
        console.log('create Reaction: addReaction', result);

        return result;
    };


    /**
     * 4. Save Reaction
     */
    const ReactionTypes = {
        Downvote : 'DOWNVOTE',
        Upvote : 'UPVOTE'
    }

    let saveReaction = async () => {
        try {
            await getUserProfile();

            await addReactionRequest({
                profileId: profile.id,
                reaction: ReactionTypes.Upvote,
                publicationId: '0x0199aa-0x02'
                }
            )

            console.log('add reaction: sucess');

        } catch (err) {
            console.log('error reacting ', err)
        }
    }


</script>


<div class="main">

    <div class="btn-div">
        {#if !isConnected}
            <button on:click="{connect}" class="btn">Connect  Wallet</button>
        {:else}
            <div>
                <b>Connected Address :</b> {address}
            </div>
            <div>
                {#if !accessTokenFromLens}
                    <button on:click={getAccessTokenFromLens} class="btn">Get Access Token</button>
                {:else}
                    <b>Access Token From Lens:</b>  {accessTokenFromLens}
                    <div class="btn-div">
                        <button on:click={getUserProfile} class="btn">First Get User Profile</button>
                        <button on:click={saveReaction} class="btn">Add Upvote</button>
                    </div>
                {/if}
            </div>
        {/if}
    </div>
</div>

<style>
    .main{
        height: 100vh;
    }

    .btn-div{
        display: flex;
        justify-content: center;
        align-items: center;
        height: 100%;
        flex-direction: column;
        gap: 48px;
    }

    .btn {
        display: inline-block;
        outline: 0;
        border: 0;
        cursor: pointer;
        background: #000000;
        color: #FFFFFF;
        border-radius: 8px;
        padding: 14px 24px 16px;
        font-size: 18px;
        font-weight: 700;
        line-height: 1;
        transition: transform 200ms, background 200ms;
    }

</style>
