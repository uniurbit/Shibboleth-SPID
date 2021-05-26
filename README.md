# Shibboleth-SPID
Guida per l'integrazione dell'autenticazione SPID con un IDP Shibboleth

## Requisiti del sistema
Dal 1 Marzo 2021 le pubbliche amministrazioni devono supportare l'autenticazione via SPID.
In generale, il sistema deve permettere all'utente di scegliere se autenticarsi presso l'identity provider, IDP, istituzionale o su uno o più IDP esterni (il primo sistema di autenticazione esterno considerato è quello di SPID). 
La soluzione permette di:

* utilizzare un IDP esterno per l'autenticazione dell'utente
* integrare gli attributi dell'IDP esterno con quelli istituzionali
* non modificare la configurazione dei service provider, SP, che si rivolgono solo all'IDP istituzionale
* utilizzare Shibboleth, senza sviluppare plugin difficili da mantenere.

## Soluzione individuata
L'idea iniziale del flusso di autenticazione è stata presa dalla guida pubblicata dall'[Università di Padova](https://github.com/bpnx/external-idp-autentication/blob/master/HOW_IT_WORKS-it.md) ed è stata completata, resa funzionante e adattata alla realtà dell'Università degli Studi di Urbino.

Questa guida è stata pensata per essere il più generale possibile e applicabile a una qualsiasi installazione di un Identity Provider di Shibboleth.

Termini utilizzati nella guida:

* **IDPint**: IDP istituzionale
* **IDPext**: IDP esterno (nel caso implementativo gli Identity Provider di SPID)
* **SPidpint**: SP installato nello stesso server di IDPint collegato a IDPext


###  Il flusso
Il flusso di esecuzione può essere sintetizzato come segue:

1. l'utente si connette ad un servizio SP1 di IDPint
2. il servizio SP1 lo reinderizza a IDPint
3. l'utente può scegliere se autenticarsi con IDPint, autenticazione classica, o con IDPext:
	* se l'autenticazione scelta è con IDPint il flusso è quello classico 
    * se si sceglie l'autenticazione esterna:
    	1. l'utente è indirizzato, attraverso SPidpint, a IDPext e si autentica
        2. si ritorna a SPidpint che trasmette gli attributi esterni a IDPint 
        3. gli attributi sono riconciliati con quelli dell'IDPint, implementando un meccanismo di gestione delle identità multiple 
4. l'utente autenticato ritorna su SP1, con gli attributi richiesti

###  Diagramma del flusso
<p align="center">
	<img alt="Diagramma flusso autenticazione Shibboleth SPID" src="https://github.com/uniurbit/Shibboleth-SPID/blob/main/diagramma-flusso-autenticazione-shibboleth-spid.jpeg"/>
</p>

###  Diagramma interazione componenti
<p align="center">
	<img alt="Diagramma interazione autenticazione Shibboleth SPID" src="https://github.com/uniurbit/Shibboleth-SPID/blob/main/diagramma-interazione-autenticazione-shibboleth-spid.jpeg"/>
</p>

## Requisiti Software
- Debian 10
- Apache/2.4.38
- Utente sudo
- Shibboleth IDP = 3.4.x o 4.0.1
	* 3.4.x http://shibboleth.net/downloads/identity-provider/latest3/
	* 4.0.1 http://shibboleth.net/downloads/identity-provider/4.0.1/
- Shibboleth SP = 3.0.4
	* 3.0.4 via apt
- Shibboleth EDS = 1.2.x
	* 1.2.x https://shibboleth.net/downloads/embedded-discovery-service/latest/shibboleth-embedded-ds-1.2.2.tar.gz
- spid-testenv2 (docker)
	* https://github.com/italia/spid-testenv2/releases		
- spid-saml-check (docker)
	* https://github.com/italia/spid-saml-check/releases 
- xmlsectool (richiede JRE >= 11)
	* http://shibboleth.net/downloads/tools/xmlsectool/


## SPidpint
### Configurare l'ambiente
Definire le costanti `APACHE_LOG`, `SHIB_SP` e `SHIBD_LOG` in `/etc/environment`

- nano `/etc/environment`
    
    ```
    APACHE_LOG=/var/log/apache2
    SHIB_SP=/etc/shibboleth
    SHIBD_LOG=/var/log/shibboleth-sp
    ```
    
- source `/etc/environment`
- mkdir `/var/log/shibboleth-sp`
- chown `_shibd: /var/log/shibboleth-sp`

### Installazione Shibboleth SP
1. Diventare root

    ```
    sudo su -
    ```

2. Installazione Shibboleth SP

    ```
    apt-get update
    apt install apache2 libapache2-mod-shib2 libapache2-mod-php ntp --no-install-recommends
    ```

3. Shibboleth SP è installato in `/etc/shibboleth`

4. **Attenzione**: se Shibboleth IDP è già installato ed è stato fatto un link simbolico dei log in `/var/log`,    
l'installazione di Shibboleth SP sembra non leggere correttamente le variabili d'ambiente precedentemente impostate sovrascrivendo i permessi sulla cartella di Shibboleth IDP esistente `/var/log/shibboleth/` di fatto IDP non sarà più in grado di scrivere.

	4.1 Verificare permessi `/var/log/shibboleth` e `/opt/shibboleth-idp/logs` devono appartenere a `jetty`  
    4.2 Verificare che in `/opt/shibboleth-idp` non siano presenti cartelle con permessi impostati a `_shibd`, in caso contrario concederli a `jetty`
    4.3 Nel file `/etc/shibboleth/shibd.logger` aggiornare i percorsi alla cartella log di Shibboleth SP `/var/log/shibboleth-sp` per i file: `shibd.log`, `shibd_warn.log`, `signature.log`, `transaction.log`
        
    4.4 Riavviare Shibboleth SP `/etc/init.d/shibd restart` e Shibboleth IDP `/etc/init.d/jetty stop` > `/etc/init.d/jetty start`

5. Per il salvataggio di metadati statici creare cartella `/etc/shibboleth/metadata` e concederla a `_shibd`, generalmente i metadati dinamici vengono salvati in `/var/cache/shibboleth`

### Generare metadata per SPID
- Spostarsi in `/etc/shibboleth/cert`
- Generare il certificato e la chiave

```
    /usr/sbin/shib-keygen -e https://idp.it/Shibboleth.sso -h idp.it -o .
```

il risultato produce i file: `sp-key.pem` e `sp-cert.pem`

- Generare i metadati

```
/usr/bin/shib-metagen -c sp-cert.pem -h idp.it -e https://idp.it/Shibboleth.sso -L -f urn:oasis:names:tc:SAML:2.0:nameid-format:transient -o "Nominativo Ente" -u "https://sitoente.it" > metagen-metadata.xml
```
- `cp -v metagen-metadata.xml updated-metadata.xml`
- apportare le seguenti modifiche al file `updated-metadata.xml` per rispettare le regole imposte da SPID 


	1. Aggiunta dell'attributo `ID` nell'elemento `EntityDescriptor`. Si tratta di un valore del tipo [`NCName`](https://www.w3.org/TR/xmlschema-2/#NCName). Meno formalmente è sufficiente una stringa con un `_` iniziale e 43 valori `[0-9a-f]` casuali
    
    2. Aggiunta namespace `xmlns:spid="https://spid.gov.it/saml-extensions"` nell'elemento `EntityDescriptor`
	
    3. L'elemento `SPSSODescriptor` deve avere gli attributi `AuthnRequestsSigned` e `WantAssertionsSigned`, con il valore `"true"`
	
    4. Eliminare tutti i nodi `AssertionConsumerService` e lasciare solo quelli con binding accettati da SPID (HTTP-POST o HTTP-Redirect): rimane solo quello con `urn:oasis:names:tc:SAML:2.0:bindings:HTTP-POST`
    	4.1 modificare l'indice a zero (il tool inizia con l'indice 1) ed aggiungere l'attributo: `isDefault="true"`
	
    5. Aggiunta attributo `use="signing"` all'elemento `KeyDescriptor`
	
    6. Aggiunta elementi `AttributeConsumingService` nell'`SPSSODescriptor`
     ```
     <SPSSODescriptor .....>
     ...
        <md:AttributeConsumingService index="0">
            <md:ServiceName xml:lang="it">SP-ENTE SPID</md:ServiceName>
            <md:ServiceDescription xml:lang="it">Descrizione SP per accesso con SPID</md:ServiceDescription>
            <md:RequestedAttribute Name="name" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="fiscalNumber" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="familyName" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="spidCode" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="gender" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="dateOfBirth" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="countyOfBirth" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="idCard" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="registeredOffice" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="email" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="digitalAddress" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="ivaCode" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="placeOfBirth" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="companyName" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="mobilePhone" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="address" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
            <md:RequestedAttribute Name="expirationDate" NameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
        </md:AttributeConsumingService>
    </md:SPSSODescriptor>
    ```
	7. Aggiunta delle informazioni riguardanti la persona di supporto (esempio per Ente della Pubblica Amministrazione)
    ```
        <md:Organization>
            <md:OrganizationName xml:lang="it">Nome Ente</md:OrganizationName>
            <md:OrganizationDisplayName xml:lang="it">Nome visualizzato</md:OrganizationDisplayName>
            <md:OrganizationURL xml:lang="it">URL del sito dell'Ente</md:OrganizationURL>
        </md:Organization>
        <md:ContactPerson contactType="other">
            <md:Extensions>
                <spid:IPACode xmlns:spid="https://spid.gov.it/saml-extensions">IPA</spid:IPACode>
                <spid:Public xmlns:spid="https://spid.gov.it/saml-extensions"/>
            </md:Extensions>
            <md:EmailAddress>email di gruppo</md:EmailAddress>
        </md:ContactPerson>
	```
    8. Convertire i certificati .crt del dominio in .pem
    ```
    openssl x509 -in idp_it.crt -out idp_it.pem -outform PEM
    ```
	9. Firmare i metadati con *xmlsectool* 
    ```
	./xmlsectool.sh --sign --inFile /etc/shibboleth/cert/updated-metadata.xml --outFile /etc/shibboleth/cert/sp-signed-metadata.xml --keyFile /etc/ssl/private/idp_it.key --certificate /etc/ssl/certs/idp_it.pem --referenceIdAttributeName ID
    ```
    10. Verificare la signature dei metadati
    ```
	./xmlsectool.sh --verifySignature --inFile /etc/shibboleth/cert/sp-signed-metadata.xml --certificate /etc/ssl/certs/idp_it.pem --referenceIdAttributeName ID
	```
    11. `sp-signed-metadata.xml` è da pubblicare su un sito https il cui url è da comunicare da AgID.

#### Convalida metadati SPID
AgID ha sviluppato uno strumento [spid-saml-check](https://github.com/italia/spid-saml-check/releases) per validare i metadati, grazie a questo possiamo verificare che siano soddisfatti tutti i requisiti imposti da SPID e aggiornati in base alle variazioni della normativa.

Tramite interfaccia grafica è possibile indicare la URL dei metadati e procedere alla validazione.
Il dettaglio in ogni punto aiuterà a capire il tipo di errore.

### Mappa attributi rilasciati da IDPext - configurazioni per SPID
È fondamentale mappare in SPidpint gli attributi ricevuti dall'IDPext per poterli passare successivamente a IDPint.

Nel file `/etc/shibboleth/attribute-map.xml` inserire la seguente parte:
```xml
<Attributes xmlns="urn:mace:shibboleth:2.0:attribute-map" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
    <Attribute name="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent" id="persistent-id">
        <AttributeDecoder xsi:type="NameIDAttributeDecoder" formatter="$NameQualifier!$SPNameQualifier!$Name" defaultQualifiers="true"/>
    </Attribute>
    <!-- SPID Custom Attributes (uri) -->
    <Attribute name="address" id="ADDRESS" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="companyName" id="COMPANYNAME" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="countyOfBirth" id="COUNTYOFBIRTH" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="dateOfBirth" id="DATEOFBIRTH" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="digitalAddress" id="DIGITALADDRESS" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="email" id="EMAIL" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="expirationDate" id="EXPIRATIONDATE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="familyName" id="FAMILYNAME" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri" />
    <Attribute name="fiscalNumber" id="FISCALNUMBER" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="gender" id="GENDER" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="idCard" id="IDCARD" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="ivaCode" id="IVACODE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="mobilePhone" id="MOBILEPHONE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="name" id="NAME" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri" />
    <Attribute name="placeOfBirth" id="PLACEOFBIRTH" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="registeredOffice" id="REGISTEREDOFFICE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri"/>
    <Attribute name="spidCode" id="SPIDCODE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:uri" />

    <!-- SPID Custom Attributes (basic) -->
    <Attribute name="address" id="ADDRESS" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="companyName" id="COMPANYNAME" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="countyOfBirth" id="COUNTYOFBIRTH" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="dateOfBirth" id="DATEOFBIRTH" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="digitalAddress" id="DIGITALADDRESS" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="email" id="EMAIL" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="expirationDate" id="EXPIRATIONDATE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="familyName" id="FAMILYNAME" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic" />
    <Attribute name="fiscalNumber" id="FISCALNUMBER" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="gender" id="GENDER" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="idCard" id="IDCARD" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="ivaCode" id="IVACODE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="mobilePhone" id="MOBILEPHONE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="name" id="NAME" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic" />
    <Attribute name="placeOfBirth" id="PLACEOFBIRTH" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="registeredOffice" id="REGISTEREDOFFICE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic"/>
    <Attribute name="spidCode" id="SPIDCODE" nameFormat="urn:oasis:names:tc:SAML:2.0:attrname-format:basic" />
</Attributes>
```


## EDS
### Installazione di un Embedded Discovery Service
Avendo a disposizione più gestori di Identità Digitale (IDPext) si è deciso di installare un EDS.

1. `sudo su -`
2. `cd /usr/local/src`
3. `wget https://shibboleth.net/downloads/embedded-discovery-service/latest/shibboleth-embedded-ds-1.2.2.tar.gz -O shibboleth-eds.tar.gz`
4. `tar xzf shibboleth-eds.tar.gz`
5. `cd shibboleth-embedded-ds-1.2.2`
6. `sudo apt install make ; make install`
7. Aggiungere le configurazioni del Discovery Service su apache2
   `mv /etc/shibboleth-ds/shibboleth-ds.conf /etc/apache2/conf-available/shibboleth-ds.conf`    
8. Abilitare la configurazione del Discovery Service su apache2
    `a2enconf shibboleth-ds.conf`
9. Riavviare Apache
    `systemctl restart apache2.service`
    
### Configurazione
#### EDS
Modificare il file `/etc/shibboleth-ds/idpselect_config.js`
```javascript
this.returnWhiteList = [ "^https:\/\/idp\.it\/Shibboleth\.sso\/Login.*$"];
this.autoFollowCookie = 'choosenIdp';

```
Si è deciso di personalizzare solamente queste due proprietà mantenendo il più possibile lo standard, per approfondire, è possibile consultare la [wiki](https://wiki.shibboleth.net/confluence/display/EDS10/)

#### Shibboleth SP

La configurazione della `SessionInitiator` in `shibboleth2.xml` deve essere rivista per dialogare correttamente con EDS e in particolare con IDPext.

Per semplicità è suggerito impiegare il tag [`<SSO>`](https://wiki.shibboleth.net/confluence/display/SP3/SSO) sarà Shibboleth a creare la struttura di `<SessionInitiator>` a runtime tipicamente sul percorso `/Login`.

Nel caso specifico SPID, sono necessari ulteriori attributi per la richiesta. 

E' necessario definire il tag `<SessionInitiator>` (documentazione  [SessionInitiator](https://wiki.shibboleth.net/confluence/display/SP3/SessionInitiator), [SAMLDS+SessionInitiator](https://wiki.shibboleth.net/confluence/display/SP3/SAMLDS+SessionInitiator) )di tipo `type"Chaining"`.
Ottenendo in cascata SAML2, SAMLDS.

```xml
     <SessionInitiator type="Chaining"
                Location="/Login"
                isDefault="true"
                outgoingBinding="urn:oasis:names:tc:SAML:profiles:SSO:request-init"
                isPassive="false"
                signing="true">
            <SessionInitiator type="SAML2" acsIndex="0" acsByIndex="true">
                <samlp:AuthnRequest xmlns:samlp="urn:oasis:names:tc:SAML:2.0:protocol"
                    xmlns:saml="urn:oasis:names:tc:SAML:2.0:assertion" ID="SPID" Version="2.0" IssueInstant="2021-01-01T00:00:00Z"
                    AttributeConsumingServiceIndex="0" ForceAuthn="true">
                    <saml:Issuer Format="urn:oasis:names:tc:SAML:2.0:nameid-format:entity" NameQualifier="https://idp.it/Shibboleth.sso">https://idp.it/Shibboleth.sso</saml:Issuer>
                    <samlp:NameIDPolicy Format="urn:oasis:names:tc:SAML:2.0:nameid-format:transient"/>
                </samlp:AuthnRequest>
            </SessionInitiator>
            <SessionInitiator type="SAMLDS" URL="https://idp.it/shibboleth-ds/index.html"/>
        </SessionInitiator>     
```

## IDPint

### Configurazione con MFA
La tipologia di autenticazione utilizzata per questa soluzione è la MFA, `Multifactor Authentication`.
Per l'implementazione di questa soluzione è necessario eseguire le seguenti modifiche:

* `/opt/shibboleth-idp/idp.properties`: aggiornare `idp.auth.flows=MFA`
* `/opt/shibboleth-idp/conf/authn/general-auth.xml`: 
Ordinare i bean nel file spostandoli nello stesso ordine in cui sono definiti nell'MFA, quindi `Password`, `RemoteUser`, `MFA`. 
Inserire il bean per il flow di autenticazione MFA.

   ```xml
    <bean id="authn/MFA" parent="shibboleth.AuthenticationFlow"
                p:passiveAuthenticationSupported="true"
                p:forcedAuthenticationSupported="true">
            <property name="supportedPrincipals">
                <list>
                    <bean parent="shibboleth.SAML2AuthnContextClassRef"
                        c:classRef="urn:oasis:names:tc:SAML:2.0:ac:classes:RemoteUser" />
                    <bean parent="shibboleth.SAML2AuthnContextClassRef"
                        c:classRef="urn:oasis:names:tc:SAML:2.0:ac:classes:PasswordProtectedTransport" />
                    <bean parent="shibboleth.SAML2AuthnContextClassRef"
                        c:classRef="urn:oasis:names:tc:SAML:2.0:ac:classes:Password" />
                    <bean parent="shibboleth.SAML1AuthenticationMethod"
                        c:method="urn:oasis:names:tc:SAML:1.0:am:password" />
                </list>
            </property>
        </bean>
   ```
   
* `/opt/shibboleth-idp/conf/authn/mfa-auth-config.xml`:

	```xml
	<util:map id="shibboleth.authn.MFA.TransitionMap">
        <entry key="">
      	  <bean parent="shibboleth.authn.MFA.Transition" p:nextFlow="authn/Password" />
    	</entry>
    	<entry key="authn/Password">
        	<bean parent="shibboleth.authn.MFA.Transition">
            	<property name="nextFlowStrategyMap">
                	<map>
                    	<entry key="ChooseMethodB" value="authn/RemoteUser" />
                	</map>
            	</property>
        	</bean>
    	</entry>
    	<entry key="authn/RemoteUser"  >
        	<bean parent="shibboleth.authn.MFA.Transition"/>
    	</entry>
	</util:map>
	```

* `/opt/shibboleth-idp/views/login.vm`: inserire il [pulsante SPID](#implementazione-pulsante-spid) e settare il tag `name` con l'evento `_eventId_ChooseMethodB`

* `/opt/shibboleth-idp/conf/authn/authn-events-flow.xml`:

	```xml
	<end-state id="ChooseMethodB" />
    	<global-transitions>
        	<transition on="ChooseMethodB" to="ChooseMethodB" />
    	</global-transitions>

	```

**Terminate le modifiche** per applicarle è necessario:

* ricompilare il container eseguendo il file `bin/build.sh`
* riavviare i servizi:
  ```
  # stop di apache e jetty
  /etc/init.d/apache2 stop
  /etc/init.d/jetty stop
  
  # riavvio servizio di ldap (solo se si ha l'instanza di ldap nel medesimo server)
  service slapd restart
  
  # avvio di apache e jetty
  /etc/init.d/apache2 start
  /etc/init.d/jetty start
  ```

Per terminare occorre proseguire in sezione [configurazione comune ad entrambi i flussi di autenticazione](#configurazione-comune-ad-entrambi-i-flussi-di-autenticazione).

#### Problematiche dell'autenticazione MFA
*In questa sezione si riporta una problematica che è stata riscontrata con MFA e una installazione di IDP Shibboleth compilato con il modulo per l'autenticazione su RADIUS, invece dello standard LDAP.*

L'installazione custom di IDP Shibboleth e questa tipologia di autenticazione non sono compatibili.
In particolare, il `Principal`, che rappresenta il risultato del flusso di autenticazione, deve essere della classe `Radius.jaas.RadiusPrincipal` per poter essere gestito dal plugin per RADIUS.
Con la configurazione MFA invece: 

1. viene attivato il flusso `MFA`
2. si passa sul flusso di autenticazioen di default `Password`
3. si autentica su RADIUS e si conclude il flusso di autenticazione `Password`
4. il flusso MFA, per terminare, effettua l'operazione di merge dei Principal e ritorna un oggetto di tipo `AuthenticationResultPrincipal` (classe [FinalizeMultiFactorAuthentication](https://www.codota.com/code/java/classes/net.shibboleth.idp.authn.impl.FinalizeMultiFactorAuthentication)) che produce un errore di cast con la classe `Radius.jaas.RadiusPrincipal` (da `AuthenticationResultPrincipal` a `Radius.jaas.RadiusPrincipal`)

A questa problematica non è stata trovata una soluzione ma si è proceduto con i [flussi di autenticazione Password e RemoteUser](#Configurazione-con-i-flussi-password-e-remoteuser) nell'IDP custom. La configurazione MFA è stata testata per una installazione IDP Shibboleth standard.

### Configurazione con i flussi Password e RemoteUser
Mappatura Realm nei numeri nelle configurazione di IDPint</a>La tipologia di autenticazione utilizzata per questa soluzione è Password e RemoteUser
Per l'implementazione di questa soluzione è necessario eseguire le seguenti modifiche (tutti i percorsi partono dalla home di IDPint, nel caso di Urbino `/opt/shibboleth-idp`):

* `conf/idp.properties`: cambiare i flussi di autenticazione considerati `idp.authn.flows=Password|RemoteUser` (non vanno spazi frale parole)
* `conf/authn/general-authn.xml`: ordinare i bean mettendo prima password e poi remoteuser, l'ordine in cui sono scritti è rilevante
* `views/login.vm`: configurare il link di evento per l'autenticazione con SPID
  
  ```
  #if ($authenticationContext.isAcceptable($extFlow) and $extFlow.apply(profileRequestContext))
      <div class="form-element-wrapper">
         <button class="form-element form-button" type="submit" name="_eventId_$extFlow.getId()">
            #springMessageText("idp.login.$extFlow.getId().replace('authn/','')", $extFlow.getId().replace('authn/',''))
         </button>
      </div>
  #end

  ```
  
 * `views/login.vm`: inserire il [pulsante SPID](#implementazione-pulsante-spid) e per ogni tag, impostare `href` e `views/onclick`

Per terminare occorre proseguire in sezione [configurazione comune ad entrambi i flussi di autenticazione](#configurazione-comune-ad-entrambi-i-flussi-di-autenticazione).

### Configurazione comune ad entrambi i flussi di autenticazione
Di seguito i passi della configurazione comuni ad entrambe le configurazioni:

* `web.xml`: occorre inserire del codice che permette all'IDPint di riconoscere un headers di Apache come elemento per l'autenticazione. Per modificare questo file, solo la prima volta, occorre copiare `/opt/shibboleth-idp/dist/webapp/WEB-INF/web.xml` in `/opt/shibboleth-idp/edit-webapp/WEB-INF/web.xml`, modificare quest'ultimo file e al termine eseguire il build

	```
	<servlet>
        <servlet-name>RemoteUserAuthHandler</servlet-name>
        <servlet-class>net.shibboleth.idp.authn.impl.RemoteUserAuthServlet</servlet-class>
        <!-- SPID -->
        <init-param>
           <!-- permette di riconoscere gli headers -->
            <param-name>checkHeaders</param-name>
            <param-value>idpext-Id-value</param-value>
           <!-- permette di riconoscere gli headers -->
        </init-param>
        <!-- END SPID -->
        <load-on-startup>2</load-on-startup>
    </servlet>
	```

`idpext-Id` è il nome dato alla variabiale nelle configurazioni di apache per passare il codice fiscale da SPID a IPDint con un header. Esso contine il principal con il quale l'IDPint procede al rilascio degli attributi con LDAP. L'unico attributo di SPID che permette di ricercare i valori da LDAP è il codice fiscale di SPID fiscalNumber (`TINIT-codice fiscale`). 
Di seguito i passi necessari per permettere il rilascio degli attributi istituzionali anche con l'autenticazione SPID, solo se l'utente possiede delle credenziali per IDPint.

*  `/opt/shibboleth-idp/conf/attribute-resolver.xml`: 

	```
  	<resolver:DataConnector id="myLDAP" xsi:type="dc:LDAPDirectory"
        ldapURL="%{idp.attribute.resolver.LDAP.ldapURL}"
        baseDN="%{idp.attribute.resolver.LDAP.baseDN}"
        principal="%{idp.attribute.resolver.LDAP.bindDN}"
        principalCredential="%{idp.attribute.resolver.LDAP.bindDNCredential}"
        useStartTLS="%{idp.attribute.resolver.LDAP.useStartTLS:true}"
        maxResultSize="0"
        noResultIsError="false"
        >


        <dc:FilterTemplate>
          <![CDATA[
                #set ($arrPrincipal = $resolutionContext.principal.split("@"))
                (&(|(uid=$arrPrincipal.get(0))(codiceFiscale=$arrPrincipal.get(0)))(imRealm=$arrPrincipal.get(1)))
            ]]>
        </dc:FilterTemplate>
    </resolver:DataConnector>
	```

* `/opt/shibboleth-idp/conf/c14n/simple-subject-c14n-config.xml`: pulizia del principal ricevuto da SPID nel caso specifico TINIT-codicefiscale (attributo `fiscalNumber`) al fine di rimuovere la prima parte TINIT- e ottenere il solo codice fiscale da usare come identificativo della persona in LDAP

```
 <!-- Simple transforms to apply to username after authentication. -->
    <util:constant id="shibboleth.c14n.simple.Lowercase" static-field="java.lang.Boolean.TRUE"/>
    <util:constant id="shibboleth.c14n.simple.Uppercase" static-field="java.lang.Boolean.FALSE"/>
    <util:constant id="shibboleth.c14n.simple.Trim" static-field="java.lang.Boolean.TRUE"/>

    <!-- Apply any regular expression replacement pairs after authentication. -->
    <util:list id="shibboleth.c14n.simple.Transforms">
    <bean parent="shibboleth.Pair" p:first="^(.+)-(.+), (<valore_cookie>)$" p:second="valore voluto" />
    </util:list>
```

**Terminate le modifiche** per applicarle è necessario:

* ricompilare il container eseguendo il file `bin/build.sh`
* riavviare i servivi:
  
  ```
  # stop di apache e jetty
  /etc/init.d/apache2 stop
  /etc/init.d/jetty stop
  
  # riavvio servizio di ldap (solo se si ha l'instanza di ldap nel medesimo server)
  service slapd restart
  
  # avvio di apache e jetty
  /etc/init.d/apache2 start
  /etc/init.d/jetty start
  ```

### Implementazione pulsante SPID
Configurare il link dell'evento autenticazione SPID nel file `/opt/shibboleth-idp/views/login.vm` 

AgID ha definito un template dell'interfaccia grafica dell'autenticazione SPID, il bundle è presente a questo indirizzo : https://github.com/italia/spid-sp-access-button/   

E' stato scelto il *button-medium-get*.

1. Importare `css` / `js` / `svg` in una cartella pubblica servita da apache
2. Prelevare il template *button-medium-get*
3. Impostare i percorsi alle risorse `css` / `js` / `svg` 
4. Impostare per ogni tag `<a>` SPID l’`href`, e `onclick`

> `{{EVENTO}}` andrà opportunamente sostituito a seconda del flusso authn configurato

>- MFA : `ChooseMethodB` 
>- Password|RemoteUser : `authn/RemoteUser`

```html
<a href="#" class="italia-it-button italia-it-button-size-m button-spid" spid-idp-button="#spid-idp-button-medium-get" aria-haspopup="true" aria-expanded="false">
    <span class="italia-it-button-icon"><img src="/spid-sp-access-button/img/spid-ico-circle-bb.svg" onerror="this.src='/spid-sp-access-button/img/spid-ico-circle-bb.png'; this.onerror=null;" alt="" /></span>
    <span class="italia-it-button-text">Entra con SPID</span>
</a>
<div id="spid-idp-button-medium-get" class="spid-idp-button spid-idp-button-tip spid-idp-button-relative">
    <ul id="spid-idp-list-medium-root-get" class="spid-idp-button-menu" aria-labelledby="spid-idp">
        <li class="spid-idp-button-link" data-idp="arubaid">
            <a href="$flowExecutionUrl&_eventId_{{EVENTO}}=1#if($csrfToken)&${csrfToken.parameterName}=${csrfToken.token}#{else}#end" onclick="setSpid(this);"><span class="spid-sr-only">Aruba ID</span><img src="/spid-sp-access-button/img/spid-idp-arubaid.svg" onerror="this.src='/spid-sp-access-button/img/spid-idp-arubaid.png'; this.onerror=null;" alt="Aruba ID" /></a>
        </li>
        <li class="spid-idp-button-link" data-idp="infocertid">
            <a href="$flowExecutionUrl&_eventId_{{EVENTO}}=1#if($csrfToken)&${csrfToken.parameterName}=${csrfToken.token}#{else}#end" onclick="setSpid(this);"><span class="spid-sr-only">Infocert ID</span><img src="/spid-sp-access-button/img/spid-idp-infocertid.svg" onerror="this.src='/spid-sp-access-button/img/spid-idp-infocertid.png'; this.onerror=null;" alt="Infocert ID" /></a>
        </li>
        ...
    </ul>
</div>
```
5 . *(facoltativo)* Modificare il pulsante classico Shibboleth Accedi per uniformarlo allo stile SPID


*(se è stato installato un EDS)* Passi successivi

All'interno del file `/opt/shibboleth-idp/views/login.vm`

1. Implementare le funzioni javascript di scelta IDPext.

	1.1 `setSpid(d)` richiamato nell'`onclick` di ciascun tag `<a>` della lista IDPext, necessario a scrivere i cookie che verranno letti dal EDS per la scelta dell’IDPext, `_saml_idp`

	1.2 `setMyCookie(c)` scrittura di 2 cookie per il discovery service:
    - `_saml_idp` (nome non modificabile, [documentazione](https://wiki.shibboleth.net/confluence/display/EDS10/6.+Cookie+Usage)) 
    - `choosen_idp` riservato EDS per ricordare la scelta dell'utente, il nome dipende da quanto configurato sull'EDS in `/etc/shibboleth-ds/idpselect_config.js:this.autoFollowCookie`
    
```javascript
function setMyCookie(cookieval) {
	...  
	document.cookie="choosenIdp=1; path=/; SameSite=None; Secure";
	document.cookie="_saml_idp=" + cookieVal + ";expires=01 Jan 2100 00:00:00 UTC; path=/; SameSite=None; Secure";
};

function setSpid(d) {
	var el=$(d).parent("li").attr("data-idp");
	switch (el) {
		case "arubaid":
			setMyCookie('https://loginspid.aruba.it');
			break;
		case "infocertid":
			setMyCookie('https://identity.infocert.it');
			break;

		...
		default:
			break;
	}
};
```
### Gestione bypass flusso expiring-password per SPID
L’IDP ha un sistema di controllo scadenza password sull’utente con una dipendenza dall’attributo LDAP `shadowExpire`. 
Nel caso in cui un utente abbia effettuato l’accesso con credenziali SPID ma non possiede una identità in IDPint gli attributi di LDAP non sono risolti.

È quindi necessario intervenire al fine di configurare opportunamente un bypass assumendo di avere sempre presente in caso di login SPID, lo `spidFiscalNumber`

*  `/opt/shibboleth-idp/views/intercept/expiring-password.vm`: 

```
#if ($attributeContext.getUnfilteredIdPAttributes().get("spidFiscalNumber"))
     #parse("intercept/mySPID.vm")
     #break
#end
```

* `/opt/shibboleth-idp/views/intercept/miSPID.vm`: si implementa un template dummy che gestisce questo caso:

```
##
## Velocity Template for expiring password view
##
## Velocity context will contain the following properties
## flowExecutionUrl - the form action location
## flowRequestContext - the Spring Web Flow RequestContext
## flowExecutionKey - the SWF execution key (this is built into the flowExecutionUrl)
## profileRequestContext - root of context tree
## authenticationContext - context with authentication request information
## authenticationErrorContext - context with login error state
## authenticationWarningContext - context with login warning state
## ldapResponseContext - context with LDAP state (if using native LDAP)
## encoder - HTMLEncoder class
## request - HttpServletRequest
## response - HttpServletResponse
## environment - Spring Environment object for property resolution
## custom - arbitrary object injected by deployer
##

<!DOCTYPE html>
<html>
    <head>
        <meta charset="utf-8">
        <meta name="viewport" content="width=device-width,initial-scale=1.0">
        <title>#springMessageText("idp.title", "Web Login Service")</title>
        <link rel="stylesheet" type="text/css" href="$request.getContextPath()/css/main.css">
        <meta http-equiv="refresh" content="0;url=$flowExecutionUrl&_eventId_proceed=1">
    </head>
      
    <body>
           &nbsp;
    </body>
</html>

```


### Gestione della risposta SAML2 con SPID
Nella risposta SAML che l’IDPint rilascia all’SP1 vi è l’elemento `NameId` che viene popolato con un attributo a scelta (presente in LDAP) che può essere specificato nel file `/opt/shibboleth-idp/conf/saml-nameid.properties` dalla proprietà `idp.persistentId.sourceAttribute`.

Nell'implementazione tradizionale, esso è popolato con `uid`. Questa impostazione, se un utente entra con SPID e non possiede un’utenza all’interno di IDPint, provoca un errore nella risposta SAML poichè `NameID` non può essere rilasciato.

Per risolvere questo problema si è aggiunto nel file `/opt/shibboleth-idp/conf/saml-nameid.xml`

```
  <bean parent="shibboleth.SAML2AttributeSourcedGenerator"
            p:format="urn:oasis:names:tc:SAML:2.0:nameid-format:persistent"
            p:attributeSourceIds="#{ {'uid', 'spidFiscalNumber'} }" />
```
In questo modo, per utenti che hanno un’identità in IPDint, sia con autenticazione istituzionale che via SPID, il `NameID` è popolato con l’`uid`, mentre per gli altri utenti che entrano via SPID esso è popolato con il codice fiscale di SPID, `spidFiscalNumber`.
Questa soluzione permette di distinguere anche gli utenti e distinguere poi gli attributi presenti in un caso o nell’altro.


## Configurazioni Apache e risoluzione attributi SPID
### Comunicazione fra IDPint e SPidpint: le configurazioni di Apache
Questa implementazione si basa sul fatto che IDPint e SPidpint siano installate sullo stesso server e che comunichino tramite un server Apache.
Di seguito gli elementi fondamentali dell'implementazione adottata: 
* Apache ha il doppio ruolo di reverse proxy di IDPint ed SPidpint per l'autenticazione su IDPext. 
* La parte SPidpint scatta quando viene selezionata l'autenticazione esterna, quindi è necessario proteggere la `Location` di autenticazione del flow `authn/RemoteUser` con il software di Shibboleth SP. In questo modo, quando l'utente preme il pulsante per l'autenticazione esterna, passa inconsapevolmente sotto il controllo di SPidpint.
* Dopo un'autenticazione positiva, il controllo torna ad IDPint, che ha bisogno di ottenere il valore del `REMOTE_USER`. Per trasferire i dati relativi alla variabile di Apache `REMOTE_USER` si è scelto il modulo Apache `Headers` e configurare degli appositi header (`RequestHeader`); il flow `authn/RemoteUser` prevede di poter ricavare il valore del `REMOTE_USER` anche dagli header HTTP.
* Senza nessuna accortezza gli attributi rilasciati da IDPext vengono persi, perchè SPidpint e IDPint sono installate nello stesso server ma non dialogano fra di loro direttamente. Analogamente a quanto fatto con `REMOTE_USER`, è possibile passare gli attributi ricevuti da IDPext, tramite la direttiva `RequestHeader`, ad IDPint (bisogna ricordare che l'istanza SPidpint rende disponibili tutti gli attributi ricevuti all'istanza Apache, come variabili, seguendo le direttive dell'`attribute-map.xml`); su IDPint.

Riassumendo:

1. l'utente seleziona l'autenticazione esterna su IDPext e viene attivato lo SPidpint,
2. l'utente, grazie SPidpint e a EDS (se presente), viene direzionato verso un IDPext e si autentica
3. terminata l'autenticazione, l'utente "ritorna" al path iniziale, `/idp/Authn/RemoteUser`, proseguendo l'autenticazione `authn/RemoteUser`
4. IDPint utilizza come `REMOTE_USER` il valore di un particolare header http, creato da Apache tramite la direttiva `RequestHeader`
5. allo stesso modo del `REMOTE_USER`, gli attributi vengono passati da SPidpint e IDPint.

### La configurazione apache
Dato il funzionamento precedente è necessario utilizzare la seguente configurazione di Apache con le seguenti `Location`: 

```
Define VAR_COOKIEROLE 	rbc
Define VAR_COOKIEROLE_NAME _ria
Define VAR_USERROLE	fur 
Define VAR_ID_HEADERIDP	headeridp
  <IfModule mod_proxy.c>
    ProxyPreserveHost On
    RequestHeader set X-Forwarded-Proto "https"
    ProxyPass /idp http://localhost:8080/idp retry=5
    ProxyPassReverse /idp http://localhost:8080/idp retry=5
```

* `Location` standard per l'IDPint

	```
    <Location /idp>
        Require all granted
    </Location>
	```

* `Location` per il flusso di autenticazione esterna: la `Location` che corrisponde a `Authn/RemoteUser` è protetta da SPidpint così da far reindirizzare l'utente su IDPext

	```
    <Location "/idp/Authn/RemoteUser">
        AuthType shibboleth
        ShibRequestSetting requireSession true
        Require shibboleth

	# role by cookie
	UnsetEnv ${VAR_COOKIEROLE}
	# final user role
	UnsetEnv ${VAR_USERROLE}	
	SetEnvIf Cookie "(^|;\ *)${VAR_COOKIEROLE_NAME}=([^;\ ]+)" ${VAR_COOKIEROLE}=$2
	SetEnvIf ${VAR_COOKIEROLE} 	"(<regex cookie>)"	${VAR_USERROLE}=$1
	# default if regex fails
	SetEnvIf ${VAR_USERROLE} 	^$		${VAR_USERROLE}=<valore default cookie>
	
	RequestHeader set ${VAR_ID_HEADERIDP} expr=%{REMOTE_USER} 
	RequestHeader append ${VAR_ID_HEADERIDP} %{${VAR_USERROLE}}e 
		
    </Location>
	```

* `Location` dopo l'autenticazione esterna: si settano come header gli attributi passatti dall'IDPext così da poterli risolvere nell'IDPint. Questo è possibile grazie all'oggetto `shibboleth.HttpServletRequest` che permette di utilizzare le variabili server in IDPint. **NOTA**: questa soluzione utilizza gli header HTTP per trasportare i dati degli attributi e del `REMOTE_USER`, che contribuiranno a ricavare il Principal Name finale. Vanno quindi utilizzate delle accortezze per evitare che un utente possa "forgiare" header per impersonare terzi: a questo scopo è fondamentale azzerare sempre tutti gli header coinvolti; inoltre è bene utilizzare nomi di header non ovvi, in modo da rendere più complicata l'impresa ad un eventuale intruso.

	```
    <Location "/idp/profile">
	
        AuthType shibboleth
        ShibRequestSetting requireSession false
        Require shibboleth

        # Le righe successive trasportano il valore degli attributi ricevuti da IDPext
        # rendendoli disponibili, come header, a IDPint
    
		RequestHeader unset header-SPIDCODE
		RequestHeader set header-SPIDCODE expr=%{base64:%{reqenv:SPIDCODE}}
	
    	RequestHeader unset header-NAME
		RequestHeader set header-NAME expr=%{base64:%{reqenv:NAME}}
	
    	RequestHeader unset header-FAMILYNAME
		RequestHeader set header-FAMILYNAME expr=%{base64:%{reqenv:FAMILYNAME}}
	
    	RequestHeader unset header-PLACEOFBIRTH
		RequestHeader set header-PLACEOFBIRTH expr=%{base64:%{reqenv:PLACEOFBIRTH}}
	
    	RequestHeader unset header-DATEOFBIRTH
		RequestHeader set header-DATEOFBIRTH expr=%{base64:%{reqenv:DATEOFBIRTH}}
	
    	RequestHeader unset header-GENDER
		RequestHeader set header-GENDER expr=%{base64:%{reqenv:GENDER}}
	
    	RequestHeader unset header-COMPANYNAME
		RequestHeader set header-COMPANYNAME expr=%{base64:%{reqenv:COMPANYNAME}}
	
    	RequestHeader unset header-REGISTEREDOFFICE
		RequestHeader set header-REGISTEREDOFFICE expr=%{base64:%{reqenv:REGISTEREDOFFICE}}
	
    	RequestHeader unset header-FISCALNUMBER
		RequestHeader set header-FISCALNUMBER expr=%{base64:%{reqenv:FISCALNUMBER}}
	
    	RequestHeader unset header-IVACODE
		RequestHeader set header-IVACODE expr=%{base64:%{reqenv:IVACODE}}
	
    	RequestHeader unset header-IDCARD
		RequestHeader set header-IDCARD expr=%{base64:%{reqenv:IDCARD}}
	
    	RequestHeader unset header-MOBILEPHONE
		RequestHeader set header-MOBILEPHONE expr=%{base64:%{reqenv:MOBILEPHONE}}
	
    	RequestHeader unset header-EMAIL
		RequestHeader set header-EMAIL expr=%{base64:%{reqenv:EMAIL}}
	
    	RequestHeader unset header-ADDRESS
		RequestHeader set header-ADDRESS expr=%{base64:%{reqenv:ADDRESS}}
	
    	RequestHeader unset header-DIGITALADDRESS
		RequestHeader set header-DIGITALADDRESS expr=%{base64:%{reqenv:DIGITALADDRESS}}
    </Location>
	```
    
```
</IfModule>
```

### Risoluzione degli attributi esterni via IDPint
Per estrarre gli header di Apache e rilasciarli come attributi è necessario richiamarli nel file `/opt/shibboleth-idp/conf/attribute-resolver.xml`, l'esempio riportato segue la sintassi di Shibboleth IDP v3.4x:

```
<resolver:AttributeDefinition id="spidPlaceOfBirth" xsi:type="ad:Script" customObjectRef="shibboleth.HttpServletRequest">
                <resolver:AttributeEncoder xsi:type="enc:SAML2String" name="spidPlaceOfBirth" friendlyName="spidPlaceOfBirth" />
                <ad:Script><![CDATA[       
                if(custom.getHeader("header-ATTRIBUTO") != null){
                        if (custom.getHeader("header-ATTRIBUTO") != ""  && custom.getHeader("header-ATTRIBUTO") != "(null)" ) {

                                var Base64 = Java.type("org.bouncycastle.util.encoders.Base64");
                                var String = Java.type("java.lang.String");
                                var str = new String(Base64.decode(custom.getHeader("header-ATTRIBUTO")));
                                spidPlaceOfBirth.addValue(str);

                         }
                }
        ]]></ad:Script>
</resolver:AttributeDefinition>

```

Dato che negli attributi SPID possono essere presenti caratteri speciali non ammessi dagli header HTTP, è stato necessario codificarli in apache in base64 (come riportato nelle configurazioni di Apache) e decodificarli in IDPint nel file `attribute-resolver.xml`.

Il codice di risoluzione è scritto in [Nashorn di Java](https://docs.oracle.com/javase/8/docs/technotes/guides/scripting/nashorn/api.html).
Inoltre:

* è stato creato uno script python in grado di generare i bean SPID senza doverli scrivere manualmente.
* a causa di un bug di Shibboleth sulla release 4.0.1, nel caso in cui un utente si autentichi con delle credenziali a cui è associato almeno un attributo tra quelli indicati negli `exportAttributes` non definito a schema (**Attenzione!** Non definito a schema, si intende non presente in LDAP, infatti se presente ma vuoto, il problema non sussiste) il `DataConnector` di LDAP fallisce con una `NullPointerException`. 
* Risolvere temporaneamente codificando come nella versione 3, in `attribute-resolver.xml` in attesa del rilascio della 4.1, anche se viene proposto un failover verso un `DataConnector` statico.
	* https://issues.shibboleth.net/jira/browse/IDP-1654
    * https://issues.shibboleth.net/jira/browse/IDP-1623
    * https://wiki.shibboleth.net/confluence/display/IDP4/ReleaseNotes

#### Definizione attributi SPID in Shibboleth IDP 4

L'unica nota di rilievo riguarda introduzione dell'[`AttributeRegistry`](https://wiki.shibboleth.net/confluence/display/IDP4/AttributeRegistryConfiguration). 

E' stato creato un file dedicato in cui sono presenti le definizioni degli attributi SPID : `/opt/shibboleth-idp/conf/attributes/spid.xml`.

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                           http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd"

       default-init-method="initialize"
       default-destroy-method="destroy">

    <bean parent="shibboleth.TranscodingRuleLoader">
    <constructor-arg>
    <list>

        <bean parent="shibboleth.TranscodingProperties">
            <property name="properties">
                <props merge="true">
                    <prop key="id">spidSpidCode</prop>
                    <prop key="transcoder">SAML2StringTranscoder</prop>
                    <prop key="saml2.name">spidCode</prop>
                    <prop key="saml2.nameFormat">urn:oasis:names:tc:SAML:2.0:attrname-format:basic</prop>
                    <prop key="displayName.en">SPID spidCode</prop>
                    <prop key="displayName.it">SPID spidCode</prop>
                    <prop key="description.en">SPID identification code</prop>
                    <prop key="description.it">SPID Codice identificativo</prop>
                </props>
            </property>
        </bean>

...

        <bean parent="shibboleth.TranscodingProperties">
            <property name="properties">
                <props merge="true">
                    <prop key="id">spidDigitalAddress</prop>
                    <prop key="transcoder">SAML2StringTranscoder</prop>
                    <prop key="saml2.name">spidDigitalAddress</prop>
                    <prop key="saml2.nameFormat">urn:oasis:names:tc:SAML:2.0:attrname-format:basic</prop>
                    <prop key="displayName.en">SPID digitalAddress</prop>
                    <prop key="displayName.it">SPID digitalAddress</prop>
                    <prop key="description.en">SPID DigitalAddress</prop>
                    <prop key="description.it">SPID Domicilio digitale</prop>
                </props>
            </property>
        </bean>
    </list>
    </constructor-arg>
    </bean>

</beans>
```

Questo deve essere importato all'interno di `/opt/shibboleth-idp/conf/attributes/default-rules.xml` altrimenti Shibboleth IDP non saprebbe applicare la codifica agli attributi da rilasciare.

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:util="http://www.springframework.org/schema/util"
       xmlns:p="http://www.springframework.org/schema/p"
       xmlns:c="http://www.springframework.org/schema/c"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
                           http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
                           http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd"

       default-init-method="initialize"
       default-destroy-method="destroy">

    <import resource="inetOrgPerson.xml" />
    <import resource="eduPerson.xml" />
    <import resource="eduCourse.xml" />
    <import resource="samlSubject.xml" />
    <import resource="schac.xml" />
    <import resource="spid.xml" />
</beans>

```

## Gestione identità multiple
Una persona ha una sola identità SPID ma, all'interno di un sistema di autenticazione, può avere più identità corrispondenti a ruoli diversi. Occorre implementare un meccanismo che permetta all'utente di autenticarsi con SPID e scegliere con quale ruolo entrare nel servizio selezionato.

Nelle sezioni successive si riporta un esempio applicato ad un Ateneo ma la stessa soluzione può essere riportata in diverse realtà.
Un esempio di gestione delle identità multiple in una Università è dato da una persona che possiede sia le credenziali da dipendente sia quelle da studente.


### Da credenziali di Ateneo a SPID
Un SP che supporta l’autenticazione solo con credenziali SPID, senza identità di Ateneo, deve essere in grado di effettuare la riconciliazione da identità di Ateneo a identità SPID:

- L'utente entra nell’SP la prima volta con SPID senza possedere l'identità di Ateneo
- Successivamente vengono assegnate delle identità di Ateneo per i ruoli necessari al servizio
- Quando si entrerà nell’SP via identità di Ateneo, in base al ruolo, dovrà esser associato all'identità SPID che possiede nel servizio

Questa operazione deve essere implementata lato SP e dunque non corrisponde a nessuna implementazione lato IDP.

### Da SPID a credenziali di Ateneo
L'identità SPID non corrisponde a nessun ruolo definito in Ateneo. 

Nel processo di autenticazione SPID, ove siano individuate più identità corrispondenti in Ateneo, deve essere possibile per l’utente scegliere con quale ruolo entrare.

Questa soluzione richiede diverse modifiche nell’implementazione adottata che vengono riportate in questa guida.

### Soluzione tecnica
Shibboleth SP permette, dopo l'autenticazione, di rimandare ad una URL intermedia `sessionHook` in cui è possibile fare controlli sull’utente.

Si imposta la `sessionHook` ad una pagina che procederà ad un controllo su database di Ateneo con chiave *codiceFiscale*.

- 1 identità, procedere direttamente su IDPint impostando il cookie del ruolo
- +1 identità, mostrare la scelta ruolo all’utente, tramite un pulsante verrà impostato un cookie relativo al ruolo

In Apache, alla `/idp/Authn/RemoteUser` viene recuperato il cookie con il ruolo ed inserito in una variabile d’ambiente
da pulire ogni volta e impostata secondo vincoli 

Viene cosi *appeso* al `RequestHeader` con il `REMOTE_USER` anche il ruolo scelto *(ancora valore grezzo)*

le configurazioni di apache nel file `/etc/apache2/sites_enabled/idp.conf`


#### Pagina riconciliazione identità
E' stato creato un bundle per la riconciliazione identità in *php*. 

Viene preso il `fiscalNumber` dell’autenticazione SPID come identificatore univoco della persona, tramite una query in database si ottengono i ruoli che la persona ricopre in Ateneo.

* **0** : viene automaticamente trasferito l’utente alla destinazione finale come persona SPID in quanto non ha ruoli in Ateneo 
* **1** : viene automaticamente trasferito l’utente alla destinazione finale con il suo ruolo
* **2+** : viene proposta una scelta del ruolo che imposta un cookie letto da apache. Secondo un mapping in IDPint viene infine tradotto in `realm`.

Una volta che l'utente ha deciso quale ruolo avere è fondamentale passare questa informazione all'IDPint che necessita del filtro per recuperare gli attributi da LDAP. Quest'ultimo è costituito dal `Principal` e dunque va configurato nel file `/opt/shibboleth-idp/conf/c14n/simple-subject-c14n-config.xml`. Le modifiche da apportare dipendono dalla propria struttura degli utenti:

```
 <!-- Simple transforms to apply to username after authentication. -->
    <util:constant id="shibboleth.c14n.simple.Lowercase" static-field="java.lang.Boolean.TRUE"/>
    <util:constant id="shibboleth.c14n.simple.Uppercase" static-field="java.lang.Boolean.FALSE"/>
    <util:constant id="shibboleth.c14n.simple.Trim" static-field="java.lang.Boolean.TRUE"/>

    <!-- Apply any regular expression replacement pairs after authentication. -->
    <util:list id="shibboleth.c14n.simple.Transforms">
    <bean parent="shibboleth.Pair" p:first="^(.+)-(.+), (<caso1>)$" p:second="valore 1 finale per filtro LDAP" />
    ...
    <bean parent="shibboleth.Pair" p:first="^(.+)-(.+), (<caso n>)$" p:second="valore n finale per filtro LDAP" />
    </util:list>
```


Guida realizzata da Francesco Buresta (francesco.buresta@uniurb.it) e Alessia Ventani (alessia.ventani@uniurb.it) per l'Università degli Studi di Urbino Carlo Bo, Febbraio 2021.
