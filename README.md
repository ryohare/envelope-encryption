# envelope-encryption

https://cashapp.github.io/2020-03-04/app-layer-encryption

Leveraging AWS KMS
	• Encryption as a service which is callable in the environment
	• Encrypted/decrypted on demand
	• Limitations
		○ 4K for symmetric key encryption
			§ Encryption is done in KMS and returned to the called
		○ Less for public key - data stored in KMS
			§ Encryption is done in KMS and returned to the called
Envelop Encryption
	• 2 keys
		○ Data key
			§ Used to encrypt the data
		○ Master key
			§ Used to encrypt the data key
	• Approach
		○ Generate new EK for each encryption operation
			§ Each encrypted JSON blob would get a key
			§ Called the data key (DK) in AWS lingo
		○ EK/DK is sent to KMS for encryption and storage
		

		
	•  Downsides
		○ Threat of downtime and overhead in using a per encryption/decryption calls to KMS
Envelope with Service Key Encryption
	• 3 keys
		○ Master Key (MK)
			§ Used to encrypt the KMS
			§ One assigned to each service, used to recall the SK
		○ Service Key (SK)
			§ Key used to encrypt all the EK/DKs
			§ These keys are managed in the service its self
				□ Service responsible for persistence
				□ So, stored in a maria DB
			§ PoC leverages google TINK library
				□ Service keys stored as tink keysets
			
		○ Encryption Key / Data Encryption Key (EK/DK)
			§ Key used to encrypt each data element - json blobs at this point


		
	• Each service is a assigned a key which is used to envelop encrypt the service keys

Threat Model
	• Threat
		○ An attacker gains access to the service data store
		○ Counter Measure
			§ Proposed encryption would ensure that all data is encryped via the EK/DK which is known to the service. The attacker would have to compromise the service key either via the service itself or KMS. The attacker could also compromise the KMS master key.
		○ An attacker gains access to the IAM role of the service granting it access to the KMS
		○ Counter Measure
			§ This would grant the actor access to the KMS however they would still need to provide the KMS master key assigned to the service to unlock in the instance. This data would need to be compromised from the service itself.
		○ An attacker has access to the service code or deployment artifacts
		○ Counter Measure
			§ If an attacker were able to gain access the source code and artifacts, the generally would have the access to the KMS master key for service however they would not poses the proper IAM role to access the KMS instance to decrypt the service key.
Key Rotation Protocol
	• Add new SK and distribute to services
	• Once data retention period has expired, remove old SK, data encrypted with old SK is now unusable and safely deletable via a batch process looking for who is encrypted with that key
	• Old data recalled can have the EK/DK re-encrypted with the updated SK, extending the retention period.
HA Concerns
	• A service (collection of multiple instances of micro services) would have the same SK through regions
		○ Regions would have their own KMS key assigned to the service for faster recall
			§ Thought: IR when a KMS is compromised

DXP Application
	• Two discrete services
		○ App Engine
		○ Shadow Streams
	• Probably one KMS for each instance so it's shared between the app engine and shadow streams
	• Service keys deployed in source code (stash)
	• Would leverage vault instead of KMS
	• Rotating EKs
		○ Not possible. Scope of an intrusion is reduced due to one key per json blob
	• Rotating SKs
		○ Push a new SW version with the hard coded key
		○ Re-encryption of Eks would occur on new push, age out the old
	• Rotating KMS Key
		○ Done just through AWS
		○ If distributed KMS as square did, would have to rotate for each region
Open Issues
	• Key attestation 
Some type of way to validate the initial service key was generated securely similar to TPM key attestation processes involving digital signatures
