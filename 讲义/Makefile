# FIXME: xindy can't handle UTF-8 filenames
FILENAME := 线性代数荣誉课辅学讲义
CPNAME := LALU

LATEXMK := latexmk
LATEXMK_FLAGS := -xelatex -shell-escape -interaction=nonstopmode -file-line-error
ANSWER_BUILD_DIR := build/answers
ANSWER_MANIFEST := $(ANSWER_BUILD_DIR)/$(CPNAME).manifest.tex
ANSWER_PUBLISHER := scripts/publish-answer-manifest.pl

.PHONY: all clean

all:
	@mkdir -p $(ANSWER_BUILD_DIR)
ifdef CPNAME
	@set -eu; \
	answer_tmp=$$(mktemp "$(ANSWER_BUILD_DIR)/$(CPNAME).manifest.XXXXXX"); \
	answer_stage=$$(mktemp -d "$(ANSWER_BUILD_DIR)/$(CPNAME).answers.XXXXXX"); \
	cleanup() { \
		rm -f "$$answer_tmp" "$(CPNAME).tex" "$(CPNAME).pdf"; \
		rm -rf -- "$$answer_stage"; \
	}; \
	trap cleanup EXIT HUP INT TERM; \
	cp -p $(FILENAME).tex $(CPNAME).tex; \
	$(LATEXMK) -g $(LATEXMK_FLAGS) \
		-usepretex="\\def\\LALUAnswerManifestTempPath{$$answer_tmp}" \
		$(CPNAME).tex; \
	perl $(ANSWER_PUBLISHER) --check "$$answer_tmp"; \
	$(LATEXMK) -g $(LATEXMK_FLAGS) \
		-outdir="$$answer_stage" \
		-usepretex="\\def\\LALUAnswerManifestPath{$$answer_tmp}\\def\\LALUAnswerManifestJob{$(CPNAME)}" \
		LALU-answers.tex; \
	test -s "$(CPNAME).pdf"; \
	test -s "$$answer_stage/LALU-answers.pdf"; \
	mv "$(CPNAME).pdf" "$(FILENAME).pdf"; \
	mv "$$answer_stage/LALU-answers.pdf" LALU-answers.pdf; \
	perl $(ANSWER_PUBLISHER) --publish "$$answer_tmp" "$(ANSWER_MANIFEST)"
else
	$(LATEXMK) $(LATEXMK_FLAGS) $(FILENAME).tex
	$(LATEXMK) $(LATEXMK_FLAGS) LALU-answers.tex
endif

clean:
	rm -f *.aux *.log *.idx *.ind *.ilg *.thm *.toc *.blg *.bbl *.bcf *.out *.fls *.fdb_latexmk *.run.xml *.synctex.gz *.xdv *~ *.lof *.lot
	rm -f LALU-ans-contents.tex *.answer-manifest.pending.tex
	rm -rf -- $(ANSWER_BUILD_DIR)
ifdef CPNAME
	rm -f $(CPNAME).pdf $(CPNAME).tex
endif
