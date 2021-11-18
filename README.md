
Script to get Issues from a application in AppScan on Cloud and import in AppScan Enterprise

```bash 
#!/bin/bash
# Connect to ASOC (AppScan on Cloud) get SAST Report and import in ASE (AppScan Enterprise). This script is set to SAST Reports. Works with ASE 10.0.5.
# Before use, set variables ASOCkeyId, ASOCkeySecret, ASEhostname, ASEkeyId and ASEkeySecret.

# How to use:
# ./importIssuesfromASOCtoASE.sh <ASOC_Application_ID> <Report_Name>.xml <ASE_Application_ID>
# Example:
# ./importIssuesfromASOCtoASE.sh 5ebfa225-1234-4bb0-abcd-f3cef742be14 app_ecommerce_sast.xml 1015

############### Variable to be filled ###############
ASOCkeyId=xxxxxxxxxxxxxxxxxxxx
ASOCkeySecret=xxxxxxxxxxxxxxxxxxxx

ASEhostname=xxxxxxxxxxxxxxxxxxxx
ASEkeyId=xxxxxxxxxxxxxxxxxxxx
ASEkeySecret=xxxxxxxxxxxxxxxxxxxx
################### End Variables ###################

ASOCappId=$1
reportTitle=$2
ASEappId=$3

# Authenticate in ASOC and get ASOC Token; Input: Output
ASOCtoken=$(curl -s -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"KeyId":"'"${ASOCkeyId}"'","KeySecret":"'"${ASOCkeySecret}"'"}' 'https://cloud.appscan.com/api/V2/Account/ApiKeyLogin' | grep -oP '(?<="Token":")[^"]*')

# Generate issues security report to a specific application. Input: Application, 
reportId=$(curl -s -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' --header "Authorization: Bearer $ASOCtoken" -d '{"Configuration":{"Summary":true,"Details":true,"Discussion":true,"Overview":true,"TableOfContent":true,"Articles":true,"Advisories":true,"FixRecommendation":true,"History":true,"Coverage": true,"MinimizeDetails":true,"ReportFileType":"XML","Title":"'"$reportTitle"'","Notes":"","Locale":"en"},"OdataFilter":"","ApplyPolicies":"None"}}' "https://cloud.appscan.com/api/v2/Reports/Security/Application/$ASOCappId" | grep -oP '(?<="Id":")[^"]*')

# Wait report ready and download it
for x in {1..30}
  do
    curl -s -X GET --header 'Accept: text/xml' --header "Authorization: Bearer $ASOCtoken" "https://cloud.appscan.com/api/v2/Reports/Download/$reportId" > $reportTitle
    if [[ -s $reportTitle ]] 
	then
      break
  fi
  sleep 1
done

# Change and add some contents in XML report file to be acceptable during ASE import
scanName=$(echo $RANDOM)_asoc_export
sed -i 's/technology="Mixed"/technology="SAST" xmlExportVersion="2.4"/' $reportTitle
sed -i '3 i <fix-recommendation-group></fix-recommendation-group>' $reportTitle

# Authenticate in ASE with KeyPair and generate a sessionId to be used in next interations. Input: KeyId and KeySecret; Output: sessionID
sessionId=$(curl -s -X POST --header 'Content-Type: application/json' --header 'Accept: application/json' -d '{"keyId":"'"$ASEkeyId"'","keySecret":"'"$ASEkeySecret"'"}' "https://winappscan.lab.local:9443/ase/api/keylogin/apikeylogin" --insecure | grep -oP '(?<="sessionId":")[^"]*')

# Import ASoC XML file into ASE 
curl -s --header 'X-Requested-With: XMLHttpRequest' --header "Cookie: asc_session_id=$sessionId;" --header "Asc_xsrf_token: $sessionId" -F "scanName=$scanName" -F "uploadedfile=@$reportTitle" "https://$ASEhostname:9443/ase/api/issueimport/$ASEappId/3/" --insecure | grep -oP inprogress

reportFileName=$scanName-$reportTitle
mv $reportTitle $reportFileName
