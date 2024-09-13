SHELL := /bin/bash
CUSTOMER := $(shell basename $(shell dirname $(shell pwd) ) )
INVOICE_DATE :=  $(shell basename $(shell pwd) )

#.PHONY: timesheet.txt timesheet.html invoice.html invoice.pdf

default: invoice.pdf

clean:
	rm -fv amount.sh.inc pay.url invoice-top.html billable.sql complete.sql timesheet.txt timesheet.html email.txt email-draft.txt invoice.html invoice.pdf

amount.sh.inc: amount.sql billable.sql
	cat billable.sql amount.sql | ssh taskdb psql --tuples-only --quiet --no-align > $@
	cat $@

pay.url: amount.sh.inc timesheet.txt
	# Double-check timesheet.txt
	# Duplicate product from previous month
	# Change amount and month in item description
	# Create payment link
	# When done, put the link into pay.url
	false

invoice-top.html: invoice-top.html.in pay.url amount.sh.inc
	#false # render
	export CUSTOMER=$(CUSTOMER) INVOICE_DATE=$(INVOICE_DATE); source <(cat env.sh.inc amount.sh.inc | sed -e 's/^.*=/export &/') && envsubst < $< > $@

amount.sql: amount.sql.in env.sh.inc
	export CUSTOMER=$(CUSTOMER) INVOICE_DATE=$(INVOICE_DATE); source <(cat env.sh.inc | sed -e 's/^.*=/export &/') && envsubst < $< > $@.tmp
	mv $@.tmp $@

billable.sql: billable.sql.in env.sh.inc
	export CUSTOMER=$(CUSTOMER) INVOICE_DATE=$(INVOICE_DATE); source <(cat env.sh.inc | sed -e 's/^.*=/export &/') && envsubst < $< > $@.tmp
	mv $@.tmp $@

complete.sql: complete.sql.in billable.sql env.sh.inc
	cat billable.sql > $@.tmp
	export CUSTOMER=$(CUSTOMER) INVOICE_DATE=$(INVOICE_DATE); source <(cat env.sh.inc | sed -e 's/^.*=/export &/') && envsubst < $< >> $@.tmp
	mv $@.tmp $@

timesheet.txt: complete.sql
	ssh taskdb psql --quiet < complete.sql > $@.tmp
	mv $@.tmp $@

timesheet.html: complete.sql
	ssh taskdb psql -H --quiet < complete.sql > $@.tmp
	# avoid overflow on the right in PDF:
	sed -i -e 's/<table border="1">/<table border="1" style="font-size: 70%;">/g' $@.tmp
	mv $@.tmp $@

upload:
	rsync -rtv $(PWD) webserver:/var/www/zorg.example/html/tmp/$(CUSTOMER)
	echo https://zorg.example/tmp/$(CUSTOMER)/$(shell basename $(shell pwd))/timesheet.txt
	echo https://zorg.example/tmp/$(CUSTOMER)/$(shell basename $(shell pwd))/invoide.html
	echo https://zorg.example/tmp/$(CUSTOMER)/$(shell basename $(shell pwd))/invoice.pdf

email-draft.txt: pay.url amount.sh.inc generate-email invoice.pdf
	./generate-email | tee $@.tmp
	mv $@.tmp $@

email.txt: email-draft.txt invoice.pdf
	mutt -H email-draft.txt -a invoice.pdf
	mv email-draft.txt email.txt

invoice.html: invoice-top.html timesheet.html
	cat invoice-top.html > $@.tmp
	cat timesheet.html >> $@.tmp
	echo '<br>Thank you! <body> </html>' >> $@.tmp
	mv $@.tmp $@

invoice.pdf: invoice.html
	weasyprint invoice.html invoice.pdf

	echo Now do '"make email-draft.txt"', and then '"make email.txt"'
