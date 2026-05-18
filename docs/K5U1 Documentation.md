# K5U1 Documentation

Melvin Danielsson

## 12 Maj:

* I copied the code from the assignment into my code.  
* I created a repo.  
* I posted a ruleset.  
* I created a ci.yml file for CI Pipeline

## 14 Maj:

* I updated my ci.yml file because it had some errors  
* Tried merging a branch with failed tests and it wasn’t possible to merge  
* I looked in to how the API key works in the code  
* I realised if I change the API key value in “appsettings.json” I actually get a message that it’s “secure” even though it really isn't.   
* I realised that it says secure because the programming for checking if its secure or not seems quite lazy

## 16 Maj:

* Rootless container minska kontaktytan för attaker på grund av att det säkerstället att man har exakt så mycket behörighet som man behöver och inget mer.  
* Skapade en container registry i azure  
* Jag valde Azure Container Apps för att det är mer anpassat för just detta  
* Jag fixade azure container registry fullt, och nu kan man gå in på hemsidan via azure och det fungerar  
* Fixade keyvault  
* Jag gav roles till mig själv och container apps’  
* Fixade med managed identity och förstår nu vad det innebär

## 17 Maj:

* Skapade en CD pipeline också, så att den också gör deployment  
* Skapade en docs folder med adr fil

## 18 Maj:

* FICK ÄNTLIGEN CI/CD Pipeline att fungera (dopamin rush)  
* Testade min pipeline och tycker det är väldigt coolt att den automatiskt deployar koden till azure

