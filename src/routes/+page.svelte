<script lang="ts">
        // Define "global" variables
        let connectButton: HTMLButtonElement;
        let disconnectButton: HTMLButtonElement;
        let sendButton: HTMLButtonElement;
        let messageInputBox: HTMLInputElement;
        let receiveBox: HTMLInputElement;

        let localConnection: RTCPeerConnection | null;   // RTCPeerConnection for our "local" connection
        let remoteConnection: RTCPeerConnection | null;  // RTCPeerConnection for the "remote"

        let sendChannel: RTCDataChannel | null;       // RTCDataChannel for the local (sender)
        let receiveChannel: RTCDataChannel | null;    // RTCDataChannel for the remote (receiver)

        // Functions

        // Connect the two peers. Normally you look for and connect to a remote
        // machine here, but we're just connecting two local objects, so we can
        // bypass that step.
        function connectPeers() {
            // Create the local connection and its event listeners
            
            localConnection = new RTCPeerConnection();
            
            // Create the data channel and establish its event listeners
            sendChannel = localConnection.createDataChannel("sendChannel");
            sendChannel.onopen = handleSendChannelStatusChange;
            sendChannel.onclose = handleSendChannelStatusChange;
            
            // Create the remote connection and its event listeners
            
            remoteConnection = new RTCPeerConnection();
            remoteConnection.ondatachannel = receiveChannelCallback;
            
            // Set up the ICE candidates for the two peers
            if (remoteConnection && localConnection) {
                localConnection.onicecandidate = e => !e.candidate
                    // @ts-ignore
                    || remoteConnection.addIceCandidate(e.candidate)
                    .catch(handleAddCandidateError);

                remoteConnection.onicecandidate = e => !e.candidate
                    // @ts-ignore
                    || localConnection.addIceCandidate(e.candidate)
                    .catch(handleAddCandidateError);
                
                // Now create an offer to connect; this starts the process
                
                localConnection.createOffer()
                // @ts-ignore
                .then(offer => localConnection.setLocalDescription(offer))
                // @ts-ignore
                .then(() => remoteConnection.setRemoteDescription(localConnection.localDescription))
                // @ts-ignore
                .then(() => remoteConnection.createAnswer())
                // @ts-ignore
                .then(answer => remoteConnection.setLocalDescription(answer))
                // @ts-ignore
                .then(() => localConnection.setRemoteDescription(remoteConnection.localDescription))
                .catch(handleCreateDescriptionError);
            }
        }
        
        // Handle errors attempting to create a description;
        // this can happen both when creating an offer and when
        // creating an answer. In this simple example, we handle
        // both the same way.

        function handleCreateDescriptionError(error: any) {
        console.log("Unable to create an offer: " + error.toString());
        }

        // Handle successful addition of the ICE candidate
        // on the "local" end of the connection.

        function handleLocalAddCandidateSuccess() {
        connectButton.disabled = true;
        }

        // Handle successful addition of the ICE candidate
        // on the "remote" end of the connection.

        function handleRemoteAddCandidateSuccess() {
        disconnectButton.disabled = false;
        }

        // Handle an error that occurs during addition of ICE candidate.

        function handleAddCandidateError() {
        console.log("Oh noes! addICECandidate failed!");
        }

        // Handles clicks on the "Send" button by transmitting
        // a message to the remote peer.

        function sendMessage() {
        let message = messageInputBox.value;
        // @ts-ignore
        sendChannel.send(message);
        
        // Clear the input box and re-focus it, so that we're
        // ready for the next message.
        
        messageInputBox.value = "";
        messageInputBox.focus();
        }

        // Handle status changes on the local end of the data
        // channel; this is the end doing the sending of data
        // in this example.

        function handleSendChannelStatusChange(event: any) {
        if (sendChannel) {
            let state = sendChannel.readyState;
        
            if (state === "open") {
            messageInputBox.disabled = false;
            messageInputBox.focus();
            sendButton.disabled = false;
            disconnectButton.disabled = false;
            connectButton.disabled = true;
            } else {
            messageInputBox.disabled = true;
            sendButton.disabled = true;
            connectButton.disabled = false;
            disconnectButton.disabled = true;
            }
        }
        }

        // Called when the connection opens and the data
        // channel is ready to be connected to the remote.

        function receiveChannelCallback(event: any) {
        receiveChannel = event.channel;
        // @ts-ignore
        receiveChannel.onmessage = handleReceiveMessage;
        // @ts-ignore
        receiveChannel.onopen = handleReceiveChannelStatusChange;
        // @ts-ignore
        receiveChannel.onclose = handleReceiveChannelStatusChange;
        }

        // Handle onmessage events for the receiving channel.
        // These are the data messages sent by the sending channel.

        function handleReceiveMessage(event: any) {
            let el = document.createElement("p");
            let txtNode = document.createTextNode(event.data);
            
            el.appendChild(txtNode);
            receiveBox.appendChild(el);
            }

            // Handle status changes on the receiver's channel.
            function handleReceiveChannelStatusChange(event: any) {
            if (receiveChannel) {
                console.log("Receive channel's status has changed to " + receiveChannel.readyState);
            }
        
        // Here you would do stuff that needs to be done
        // when the channel's status changes.
        }

        // Close the connection, including data channels if they're open.
        // Also update the UI to reflect the disconnected status.

        function disconnectPeers() {

            // Close the RTCDataChannels if they're open.
            if(sendChannel) sendChannel.close();
            if(receiveChannel) receiveChannel.close();;
            
            // Close the RTCPeerConnections
            if(localConnection) localConnection.close();
            if(remoteConnection) remoteConnection.close();

            sendChannel = null;
            receiveChannel = null;
            localConnection = null;
            remoteConnection = null;
            
            // Update user interface elements
            
            connectButton.disabled = false;
            disconnectButton.disabled = true;
            sendButton.disabled = true;
            
            messageInputBox.value = "";
            messageInputBox.disabled = true;
        }
</script>

<main>
    <h1>MDN - WebRTC: Simple RTCDataChannel sample</h1>
  <p>This sample is an admittedly contrived example of how to use an <code>RTCDataChannel</code> to
  exchange data between two objects on the same page. See the
  <a href="https://developer.mozilla.org/en-US/docs/Web/API/WebRTC_API/Simple_RTCDataChannel_sample">
  corresponding article</a> for details on how it works.</p>
  
  <div class="controlbox">
    <button bind:this={connectButton} on:click={connectPeers} name="connectButton" class="buttonleft">
      Connect
    </button>
    <button bind:this={disconnectButton} on:click={disconnectPeers} name="disconnectButton" class="buttonright" disabled>
      Disconnect
    </button>
  </div>
  
  <div class="messagebox">
    <label for="message">Enter a message:
      <input type="text" name="message" bind:this={messageInputBox} placeholder="Message text" size=60 maxlength=120 disabled>
    </label>
    <button bind:this={sendButton} on:click={sendMessage} name="sendButton" class="buttonright" disabled>
      Send
    </button>
  </div>
  <div class="messagebox" id="receivebox">
    <p>Messages received:</p>
  </div>
</main>
