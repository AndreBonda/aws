# AWS
Appunti su alcuni servizi di AWS.

# Identity and Access Management (IAM)
Un *AWS Account* è un contenitore di **Identity** e **Risorse** che quando creato include un **root user**, il quale ha pieno controllo dell'account (è buona norma non operare con il root user).<br>
IAM sta per **Identity and Access Management** ed è un servizio globale che permette di gestire facilmente autenticazione e autorizzazione delle risorse.<br>
Ci sono 3 diverse tipologie di identities: **User**, **Group**, **Role**.

<img src="images/iam_overview.png" style="width: 600px;"/>

## IAM User
È un'**identità permanente** per <u>garantire accessi a lungo termine ad una persona o un'applicazione</u>. Ad esempio creare un utente per utilizzare la cli di AWS oppure un utente per ogni membro del team.<br>
Gli utenti si **autenticano** con le credenziali *user & password* oppure con *access key*.

In IAM è importante il concetto di **Principal**, ovvero una persona fisica, un’applicazione, un device o processo che si vuole autenticare in AWS. Un **Anonymous Principal** è un utente non autenticato, cioè qualcuno che accede a una risorsa AWS senza fornire credenziali.<br>
<img src="images/iam_authentication.png" style="width: 600px;"/>

## IAM Group
Sono dei <u>contenitori di IAM User</u> che ne semplificano la gestione, ad esempio il team Finance/Engineering/HR ...<br>
È possibile definire delle **Identity Policy** al gruppo, che si applicano a tutti gli utenti all'interno di quel gruppo. Non è possibile autenticarsi come un gruppo.
I gruppi non sono delle **true identity** e quindi non è possibile definire delle *Resource Policy* (non si può definire il concetto *"a questo bucket S3 può accedere questo gruppo"*)

## IAM Role
Se un IAM User è pensato per autenticare un singolo Principal, <u>IAM Role è adatto quando non sappiamo a priori chi saranno i *Principal*</u> (eventualmente anche un numero grande). Invece che autorizzare principal identificati, autorizziamo tutti i principal (non sappiamo quali e quanti) che hanno un determinato ruolo.<br>

Entità diverse possono assumere un ruolo per ottenere permessi:
- l'IAM User *mario rossi* assume il ruolo admin per fare operazioni amministrative
- un'istanza EC2 assume un ruolo per accedere ad S3

I ruoli generalmente sono utilizzati su base temporanea, quindi quando viene assunto un ruolo vengono generate delle credenziali a breve durata:
```text
1. Crei IAM Role "EC2-S3-Access"
2. Un'istanza EC2 "assume" questo ruolo
3. AWS genera credenziali temporanee valide per 1 ora
4. Dopo 1 ora, le credenziali scadono
5. AWS genera automaticamente NUOVE credenziali temporanee
```

I ruoli sono delle **real identity** a differenza dei gruppi e quindi possono essere referenziate dalle *Resource Policy*.<br>
<img src="images/iam_role.png" style="width: 600px;"/>

## IAM Policy
Le policy in AWS sono **documenti JSON** che <u>definiscono i permessi di accesso alle risorse AWS (chi può fare cosa, su quali risorse e in quali condizioni)</u>.
```json
{
 "Version": "2012-10-17",
 "Statement": [
   {
     "Effect": "Allow", // Allow or Deny
     "Action": "s3:GetObject", // What kind of action
     "Resource": "arn:aws:s3:::nome-bucket/*" // Destination resource
   }
 ]
}
```

### Identity Based Policy
Sono policy (sia *Managed* che *Inline*) che <u>assegnano permessi ad utenti, gruppi o ruoli IAM</u> e rappresentano la scelta standard e consigliata.<br>
Esempio di una policy per un IAM User:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": "arn:aws:s3:::mio-bucket-produzione"
    }
  ]
}
```

### Resource Based Policy
Policy che assegnano permessi direttamente a una risorsa AWS. La referenza viene fatta tramite ARN (ad esempio un bucket S3 permette accesso solo ad una lambda specifica).<br>
A differenza delle Identity Policy possono permettere/negare operazioni ad Identity anche esterne all’account AWS (accesso cross-account) oppure ad Anonymous Principals.<br>
Esempio di una policy per una Lambda:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowLambdaAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::123456789012:role/MyLambdaRole"
      },
      "Action": [
        "s3:GetObject",
      ],
      "Resource": "arn:aws:s3:::mio-bucket/*"
    }
  ]
}
```

### Role Policy
Quando definiamo i permessi sui ruoli abbiamo due tipi di policy.<br>

Le **Trust Policy** sono una *resource based policy* e definiscono chi può assumere il ruolo.<br>
Quando un ruolo viene assunto da un’identità, AWS genera security credentials temporanee utilizzate da tale identità (simili agli access key ma con limiti temporali). Il servizio che genera le credenziali è **Secure Token Service (STS)**.

Le **Permission Policy** sono *identity based policy* e definiscono cosa può fare chi assume un ruolo.

```
┌─────────────────────────────────────────┐
│         IAM ROLE: "AppRole"             │
├─────────────────────────────────────────┤
│                                         │
│  📋 TRUST POLICY                        │
│  ┌───────────────────────────────────┐  │
│  │ WHO can assume?                   │  │
│  │ • ec2.amazonaws.com               │  │
│  │ • lambda.amazonaws.com            │  │
│  │ • arn:aws:iam::999:user/external  │  │
│  └───────────────────────────────────┘  │
│           ↓ (if allowed)                │
│                                         │
│  🔐 STS generates temporary creds       │
│  ┌───────────────────────────────────┐  │
│  │ • AccessKeyId (ASIA...)           │  │
│  │ • SecretAccessKey                 │  │
│  │ • SessionToken                    │  │
│  │ • Expiration (1-12 hours)         │  │
│  └───────────────────────────────────┘  │
│           ↓ (identity uses creds)       │
│                                         │
│  🔑 PERMISSION POLICIES                 │
│  ┌───────────────────────────────────┐  │
│  │ WHAT can they do?                 │  │
│  │ • s3:GetObject                    │  │
│  │ • dynamodb:PutItem                │  │
│  │ • logs:CreateLogStream            │  │
│  └───────────────────────────────────┘  │
│           ↓ (if allowed)                │
│                                         │
│  ✅ Access to Resources                 │
└─────────────────────────────────────────┘
```

### How to define policies (Managed vs Inline)
Le **Managed Policy** vengono prima create e successivamente assegnate a utenti, ruoli, gruppi. Quindi riutilizzabili e una modifica alla policy si ripercuote a tutte le identity che hanno tale policy assegnata.<br>
Le Inline Policy definite direttamente per uno specifico utente, ruolo o gruppo. Utile se vogliamo garantire access/deny specifici per una identity.
