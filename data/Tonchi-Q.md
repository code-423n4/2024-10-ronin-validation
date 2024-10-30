# Beacon variable should be an immutable variable

## **Description** 
In the Beacon Proxy mechanism, the beacon variable holds the address of the Beacon contract, which in turn points to the implementation address.
	•	Marking the beacon address as immutable ensures that the proxy contract always references the same Beacon contract. Changing the beacon address could redirect the proxy to an unexpected Beacon, potentially introducing security risks and unpredictable behavior.
	•	An immutable beacon address strengthens the integrity of the proxy by preventing unauthorized or accidental updates to the implementation source.

## **Impact** 

Changing the beacon address could redirect the proxy to an unexpected Beacon, potentially introducing security risks and unpredictable behavior.

## **Recommended mitigation**
- Mark Beacon immutable if unnecessary.