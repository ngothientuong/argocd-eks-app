## Example on snapshot and restore

**Note** : 	Can't dynamically or statically allocate storage class io1, io2 in small instance size such as M5.2xlarge

		For small size storage, stick with gp2, gp3 for now.

		Have not tested statically allocate io1, io2 storage classes via creation of a pv just yet

### Procedure 

- create sc, pvc, pod

- take a snapshot of pod via volume snapshot pvc

- then delete pod , pvc

- restore a volume via volume snapshot restore claim

- spin up pod and validate

- take a snapshot of the newly created restore snapshot

