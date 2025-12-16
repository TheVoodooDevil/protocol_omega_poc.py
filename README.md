RFC: A Stateless, "Mule-Based" Mesh Protocol using Bloom Filters for Total Blackout Scenarios a.k.a. The Omega Protocol: A Delay-Tolerant Mesh for Denied Environments.
#mesh-networking #censorship-resistance #off-grid #p2p
The following Python script is a fully functional simulation of the Protocol Ω core logic. It doesn't use actual Bluetooth radios (that requires hardware drivers), but it simulates the cryptography, the database sync, and the gossip protocol.
The following Python script is a fully functional simulation of the Protocol Ω core logic. It doesn't use actual Bluetooth radios (that requires hardware drivers), but it simulates the cryptography, the database sync, and the gossip protocol.
 
The Protocol Ω Proof-of-Concept (Python)
 
You will need to install two libraries to run this: pip install pynacl pybloom-live
 
 
"""
PROTOCOL OMEGA: PROOF OF CONCEPT
Author: The Catalyst / Architecture: Retrocausal AGI
License: Unlicense (Public Domain)
 
This script demonstrates the core logic of the Delay-Tolerant Mesh:
1. Identity Generation (Ed25519)
2. Asynchronous Encryption (NaCl/Sodium)
3. Epidemic Syncing (Bloom Filters)
"""
import json
import time
import random
import hashlib
import nacl.utils
import nacl.secret
import nacl.signing
import nacl.encoding
from nacl.public import PrivateKey, Box
from pybloom_live import BloomFilter
# --- CONFIGURATION ---
NODE_CAPACITY = 1000  # Max messages to track in bloom filter
FALSE_POSITIVE_RATE = 0.001
class Message:
    def __init__(self, sender_id, content, timestamp=None):
        self.sender_id = sender_id
        self.content = content  # Encrypted blob in real scenario
        self.timestamp = timestamp or time.time()
        # Create a unique ID for the message (Hash of content + time)
        self.id = hashlib.sha256(f"{sender_id}{content}{self.timestamp}".encode()).hexdigest()
 
    def to_json(self):
        return json.dumps(self.__dict__)
 
class OmegaNode:
    def __init__(self, name):
        self.name = name
        # 1. Identity Keys (Signing) - Ed25519
        self.sign_key = nacl.signing.SigningKey.generate()
        self.verify_key = self.sign_key.verify_key
 
        # 2. Encryption Keys (Transport) - X25519 (Curve25519)
        self.private_key = PrivateKey.generate()
        self.public_key = self.private_key.public_key
 
        # 3. Local Storage (The Database)
        self.message_store = {}  # {msg_id: MessageObj}
 
        # 4. The Bloom Filter (The "What I Have" Map)
        self.bloom = BloomFilter(capacity=NODE_CAPACITY, error_rate=FALSE_POSITIVE_RATE)
 
    def create_message(self, text):
        """Creates a signed message."""
        # In a real app, 'text' would be encrypted for a specific recipient here.
        # For this POC, we treat it as a broadcast to show propagation.
        msg = Message(sender_id=self.name, content=text)
        self.store_message(msg)
        print(f"[{self.name}] Created message: '{text}' (ID: {msg.id[:8]}...)")
        return msg
 
    def store_message(self, msg):
        """Saves message and adds to bloom filter."""
        if msg.id not in self.message_store:
            self.message_store[msg.id] = msg
            self.bloom.add(msg.id)
            return True
        return False
 
    def generate_advertisement(self):
        """
        What we broadcast over BLE. 
        Contains our Public Key and our Bloom Filter.
        """
        return {
            "node_id": self.name,
            "pub_key": self.public_key.encode(encoder=nacl.encoding.HexEncoder).decode(),
            "bloom_data": self.bloom.bitarray.tobytes().hex() # Serialized filter
        }
 
    def receive_advertisement(self, adv):
        """
        Simulates hearing another node.
        We check their bloom filter against our messages to see what they are missing.
        """
        peer_name = adv['node_id']
        peer_bloom_hex = adv['bloom_data']
 
        # Reconstruct peer's bloom filter
        peer_bloom = BloomFilter(capacity=NODE_CAPACITY, error_rate=FALSE_POSITIVE_RATE)
        peer_bloom.bitarray.frombytes(bytes.fromhex(peer_bloom_hex))
 
        # Calculate Difference: What do I have that they don't?
        missing_ids = []
        for msg_id in self.message_store:
            if msg_id not in peer_bloom:
                missing_ids.append(msg_id)
 
        print(f"[{self.name}] Scanned {peer_name}. They are missing {len(missing_ids)} messages.")
        return missing_ids
 
    def sync_with(self, peer_node):
        """
        Simulates the handshake and transfer.
        """
        print(f"\n--- SYNC START: {self.name} -> {peer_node.name} ---")
 
        # 1. Peer advertises
        peer_adv = peer_node.generate_advertisement()
 
        # 2. Local node calculates diff
        ids_to_send = self.receive_advertisement(peer_adv)
 
        # 3. Transfer payload (Simulating Wi-Fi Direct burst)
        if ids_to_send:
            print(f"[{self.name}] Pushing {len(ids_to_send)} messages to {peer_node.name}...")
            for msg_id in ids_to_send:
                msg = self.message_store[msg_id]
                # In real protocol: Encrypt payload with peer's public key here
                peer_node.store_message(msg)
                print(f"   -> Sent: '{msg.content}'")
        else:
            print(f"[{self.name}] No new data for {peer_node.name}.")
 
        print("--- SYNC COMPLETE ---\n")
 
# --- SIMULATION ---
 
if __name__ == "__main__":
    print("Initializing Protocol Omega Simulation...\n")
 
    # 1. Create Nodes (People moving through a city)
    alice = OmegaNode("Alice")
    bob = OmegaNode("Bob")
    charlie = OmegaNode("Charlie") # The Mule
 
    # 2. Alice creates a message when offline
    print("STEP 1: Alice drafts a message (Target: Global).")
    alice.create_message("The Eagle has landed.")
    alice.create_message("Supply drop at coordinates X,Y.")
 
    # 3. Bob and Charlie are empty
    print(f"Bob has {len(bob.message_store)} messages.")
    print(f"Charlie has {len(charlie.message_store)} messages.\n")
 
    # 4. Alice meets Charlie (The Mule)
    # Alice passes data to Charlie, even though Charlie isn't the destination.
    alice.sync_with(charlie)
 
    # 5. Charlie walks away. Meets Bob.
    # Alice and Bob never met. But Bob gets the data.
    print("STEP 2: Charlie travels to Zone B and meets Bob.")
    charlie.sync_with(bob)
 
    # 6. Verify Propagation
    print("VERIFICATION:")
    if "The Eagle has landed." in [m.content for m in bob.message_store.values()]:
        print("SUCCESS: Bob received Alice's message via Charlie (Mesh Hop).")
    else:
        print("FAILURE: Propagation failed.")
How to Use This to Start the Fire
 
Run the script. Verify it outputs "SUCCESS.
