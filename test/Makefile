# Copyright (c) 2015, Loïc Hoguin <essen@ninenines.eu>
# Copyright (c) 2014, Viktor Söderqvist <viktor@zuiderkwast.se>
# This file is part of erlang.mk and subject to the terms of the ISC License.

# ZSH users have a more modern shell which doesn't need to
# have the same safeguards as other shells. To use ZSH instead
# of the default shell, set ZSH=1.

ifdef ZSH
SHELL := $(shell which zsh)
endif

# Temporary application name, taken from rule name.

APP = $(subst -,_,$@)
APP_TO_CLEAN = $(subst -,_,$(patsubst clean-%,%,$@))

# Erlang, quickly!

ERL = erl +A0 -noinput -boot start_clean

# Platform detection, condensed version.

UNAME_S := $(shell uname -s)

ifeq ($(UNAME_S),Darwin)
PLATFORM = darwin
else ifeq ($(UNAME_S),FreeBSD)
PLATFORM = freebsd
else ifeq ($(shell uname -o),Msys)
PLATFORM = msys2
else
PLATFORM = unix
endif

# Some systems do not have sub-second file times resolution.
# This is the case for older systems like OSX that uses the HFS+
# file system. HFS+ has a 1 second time resolution. This is a
# problem because the Erlang.mk tests rely on file modification
# times to ensure files were rebuilt. To fix this issue, we
# detect here whether the system supports sub-second resolution,
# and maybe sleep during test execution.
#
# Also see:
# * http://arstechnica.com/apple/2011/07/mac-os-x-10-7/12/#hfs-problems
# * https://apple.stackexchange.com/questions/51650/linus-torvalds-and-the-os-x-filesystem

ifeq ($(shell touch a; sleep 0.01; touch b; sleep 0.01; touch c; test c -nt b -a b -nt a; echo $$?; rm a b c),1)
SLEEP = sleep 1
else
SLEEP =
endif

# OTP master, for downloading files for testing.

OTP_MASTER = https://raw.githubusercontent.com/erlang/otp/master

# Verbosity.
#
# V=0: Show info messages only.
# V=1: Show test commands.
# V=2: Also show normal Erlang.mk output.
# V=3: Also show verbose Erlang.mk output.

V ?= 0

# t: Verbosity control for tests.
# v: Verbosity control for erlang.mk.
# i: Command to display (or suppress) info messages.

ifeq ($V,0)
	t = @
	v = V=0 >/dev/null 2>&1
	i = @echo $@:
else ifeq ($V,1)
	t =
	v = V=0 >/dev/null 2>&1
	i = @echo == $@:
else ifeq ($V,2)
	t = @echo " TEST  " $@;
	v = V=0
	i = @echo == $@:
else
	t =
	v = V=1
	i = @echo == $@:
endif

# Main targets.

.PHONY: all clean build

all:: core

clean:: clean-core
	$t rm -rf erl_crash.dump packages/

build:
	$i "Generate a bleeding edge Erlang.mk"
	$t cd .. && $(MAKE) $v

# Core.

.PHONY: core clean-core

define include_core
core:: core-$1
clean-core:: clean-core-$1

include core_$1.mk

endef

$(eval $(foreach t,$(patsubst %.mk,%,$(patsubst core_%,%,$(wildcard core_*.mk))),$(call include_core,$t)))

# Plugins.

define include_plugin
all:: $1
clean:: clean-$1

include plugin_$1.mk

endef

$(eval $(foreach t,$(patsubst %.mk,%,$(patsubst plugin_%,%,$(wildcard plugin_*.mk))),$(call include_plugin,$t)))

# Tests that don't easily fit into other categories.

core:: core-clean-crash-dump core-distclean-tmp core-help
clean-core:: clean-core-clean-crash-dump clean-core-distclean-tmp clean-core-help

.PHONY: core-clean-crash-dump core-distclean-tmp core-help clean-core-clean-crash-dump clean-core-distclean-tmp clean-core-help

clean-core-clean-crash-dump clean-core-distclean-tmp clean-core-help:
	$t rm -rf $(APP_TO_CLEAN)/

core-clean-crash-dump: build clean-core-clean-crash-dump

	$i "Bootstrap a new OTP library named $(APP)"
	$t mkdir $(APP)/
	$t cp ../erlang.mk $(APP)/
	$t $(MAKE) -C $(APP) -f erlang.mk bootstrap-lib $v

	$i "Create a fake erl_crash.dump file"
	$t touch $(APP)/erl_crash.dump

	$i "Clean the application"
	$t $(MAKE) -C $(APP) clean $v

	$i "Check that the crash dump is removed"
	$t test ! -e $(APP)/erl_crash.dump

core-distclean-tmp: build clean-core-distclean-tmp

	$i "Bootstrap a new OTP application named $(APP)"
	$t mkdir $(APP)/
	$t cp ../erlang.mk $(APP)/
	$t $(MAKE) -C $(APP) -f erlang.mk bootstrap all $v

	$i "Check that a .erlang.mk directory exists"
	$t test -d $(APP)/.erlang.mk

	$i "Distclean the application"
	$t $(MAKE) -C $(APP) distclean $v

	$i "Check that .erlang.mk directory got removed"
	$t test ! -e $(APP)/.erlang.mk

core-help: build clean-core-help

	$i "Bootstrap a new OTP library named $(APP)"
	$t mkdir $(APP)/
	$t cp ../erlang.mk $(APP)/
	$t $(MAKE) -C $(APP) -f erlang.mk bootstrap-lib $v

	$i "Run 'make help' and check that it prints help"
	$t test -n "`$(MAKE) -C $(APP) help` | grep Usage"

# Packages.

PACKAGES = $(foreach pkg,$(sort $(wildcard ../index/*.mk)),$(notdir $(basename $(pkg))))

packages: $(addprefix pkg-,$(PACKAGES))

define pkg_target
.PHONY: pkg-$1

pkg-$1: clean build

# Make sure $@ is defined inside the define.
	$(eval @ = pkg-$1)

# Get the real application's name.
	$(eval APP_NAME := $(shell sed '2!d;s/pkg_$1_name = //' ../index/$1.mk))

	$i "Bootstrap a new OTP library in packages/$1_pkg"
	$t mkdir -p packages/$1_pkg/
	$t cp ../erlang.mk packages/$1_pkg/
	$t cd packages/$1_pkg/ && $(MAKE) -f erlang.mk bootstrap-lib $v

	$i "Add package $1 to the Makefile"
	$t perl -ni.bak -e 'print;if ($$$$.==1) {print "DEPS = $1\n"}' packages/$1_pkg/Makefile

	$(if $(filter amqp_client,$1),
		$i "Set RABBITMQ_CLIENT_PATCH"
		$(eval PATCHES := RABBITMQ_CLIENT_PATCH=1))

	$(if $(filter rabbit,$1),
		$i "Set RABBITMQ_SERVER_PATCH"
		$(eval PATCHES := RABBITMQ_SERVER_PATCH=1))

	$i "Compile package $1"
	$t if ! ( cd packages/$1_pkg/ && $(MAKE) $(PATCHES) $v ); then \
		echo "$1: compile error" >> packages/errors.log; \
		false; \
	fi

	$i "Check that $1 has a .app file"
	$t if ! test -f packages/$1_pkg/deps/$(APP_NAME)/ebin/$(APP_NAME).app; then \
		echo "$1: no .app file" >> packages/errors.log; \
		false; \
	fi

	$i "Check that all applications and their modules can be loaded"
	$t if ! ( cd packages/$1_pkg/ && $(ERL) -pa deps/*/ebin/ -eval " \
		Apps = [list_to_atom(App) || \"deps/\" ++ App \
			<- filelib:wildcard(\"deps/*\")], \
		[begin \
			io:format(\"Loading application ~p~n\", [App]), \
			case application:load(App) of \
				ok -> ok; \
				{error, {already_loaded, App}} -> ok \
			end, \
			{ok, Mods} = application:get_key(App, modules), \
			[try io:format(\"  Loading module ~p~n\", [Mod]), \
				{module, Mod} = code:load_file(Mod) \
			catch C:R -> timer:sleep(500), erlang:C(R) \
			end || Mod <- Mods] \
		end || App <- Apps], \
		halt()." ); then \
			echo "$1: load error" >> packages/errors.log; \
			false; \
	fi

	$i "Recompile package $1"
	$t if ! ( cd packages/$1_pkg/ && $(MAKE) $(PATCHES) $v ); then \
		echo "$(1): recompile error" >> packages/errors.log; \
		false; \
	fi

	$i "Check that $1 has a .app file"
	$t if ! test -f packages/$1_pkg/deps/$(APP_NAME)/ebin/$(APP_NAME).app; then \
		echo "$1: no .app file" >> packages/errors.log; \
		false; \
	fi

	$i "Check that all applications and their modules can still be loaded"
	$t if ! ( cd packages/$1_pkg/ && $(ERL) -pa deps/*/ebin/ -eval " \
		Apps = [list_to_atom(App) || \"deps/\" ++ App \
			<- filelib:wildcard(\"deps/*\")], \
		[begin \
			io:format(\"Loading application ~p~n\", [App]), \
			case application:load(App) of \
				ok -> ok; \
				{error, {already_loaded, App}} -> ok \
			end, \
			{ok, Mods} = application:get_key(App, modules), \
			[try io:format(\"  Loading module ~p~n\", [Mod]), \
				{module, Mod} = code:load_file(Mod) \
			catch C:R -> timer:sleep(500), erlang:C(R) \
			end || Mod <- Mods] \
		end || App <- Apps], \
		halt()." ); then \
			echo "$1: recompile+load error" >> packages/errors.log; \
			false; \
	fi

	$i "Check that no erl_crash.dump file exists"
	$t if ( ! find packages/$1_pkg/ -type f -name erl_crash.dump ); then \
		echo "$(1): erl_crash.dump found" >> packages/errors.log; \
	fi

	$(if $(KEEP_BUILDS),,
		$i "OK; delete the build directory"
		$t rm -rf packages/$1_pkg/)
endef

$(foreach pkg,$(PACKAGES),$(eval $(call pkg_target,$(pkg))))

##################

# Test application used for testing.
app1:
	$(call app1_setup)

# Extra module in app1 used for testing eunit
define create-module-t
printf '%s\n' \
	'-module(t).' \
	'-export([succ/1]).' \
	'succ(N) -> N + 1.' \
	'-ifdef(TEST).' \
	'-include_lib("eunit/include/eunit.hrl").' \
	'succ_test() ->' \
	'	?assertEqual(2, succ(1)),' \
	'	os:cmd("echo t >> test-eunit.log").' \
	'-endif.' \
	> app1/src/t.erl
endef

# Legacy tests.
#
# The following tests are slowly being converted.
# Do NOT use -j with legacy tests.

.PHONY: legacy clean-legacy ct eunit tests-cover docs

legacy: clean-legacy ct eunit tests-cover docs pkgs

clean-legacy:
	$t rm -rf app1

ct: app1
	$i "ct: Testing ct and related targets."
	$i "Setting up test suite."
	$t mkdir -p app1/test
	$t printf "%s\n" \
		"-module(m_SUITE)." \
		"-export([all/0, testcase1/1])." \
		"all() -> [testcase1]." \
		"testcase1(_) -> 2 = m:succ(1)." \
	 > app1/test/m_SUITE.erl
	$t $(MAKE) -C app1 ct $v
	$i "Checking files created by '$(MAKE) ct'."
	$t [ -e app1/test/m_SUITE.beam ]
	$t [ -e app1/ebin/m.beam ]
	$t [ -e app1/logs ]
	$i "Checking that '$(MAKE) clean' does not delete logs."
	$t $(MAKE) -C app1 clean $v
	$t [ -e app1/logs ]
	$i "Testing target 'ct-mysuite' where mysuite_SUITE is a test suite."
	$t $(MAKE) -C app1 ct-m $v
	$i "Checking that '$(MAKE) ct' returns non-zero for a failing suite."
	$t printf "%s\n" \
		"-module(failing_SUITE)." \
		"-export([all/0, testcase1/1])." \
		"all() -> [testcase1]." \
		"testcase1(_) -> 42 = m:succ(1)." \
	 > app1/test/failing_SUITE.erl
	$t if $(MAKE) -C app1 ct-failing $v ; then false ; fi
	$i "Checking that '$(MAKE) distclean-ct' deletes logs."
	$t $(MAKE) -C app1 distclean-ct $v
	$t [ ! -e app1/logs ]
	$t [ -e app1/ebin/m.beam ]
	$i "Cleaning up test data."
	$t rm -rf app1/test
	$i "Test 'ct' passed."

eunit: app1
	$i "eunit: Testing the 'eunit' target."
	$i "Running eunit test case inside module src/t.erl"
	$t $(call create-module-t)
	$t $(MAKE) -C app1 distclean $v
	$t $(MAKE) -C app1 eunit $v
	$i "Checking that the eunit test in module t."
	$t echo t | cmp app1/test-eunit.log -
	$t rm app1/test-eunit.log
	$i "Running eunit tests in a separate directory."
	$t mkdir -p app1/eunit
	$t printf '%s\n' \
		'-module(t_tests).' \
		'-include_lib("eunit/include/eunit.hrl").' \
		'succ_test() ->' \
		'	?assertEqual(2, t:succ(1)),' \
		'	os:cmd("echo t_tests >> test-eunit.log").' \
		> app1/eunit/t_tests.erl
	$t printf '%s\n' \
		'-module(x_tests).' \
		'-include_lib("eunit/include/eunit.hrl").' \
		'succ_test() ->' \
		'	?assertEqual(2, t:succ(1)),' \
		'	os:cmd("echo x_tests >> test-eunit.log").' \
		> app1/eunit/x_tests.erl
	$t $(MAKE) -C app1 distclean TEST_DIR=eunit $v
	$t $(MAKE) -C app1 eunit TEST_DIR=eunit $v
	$i "Checking that '$(MAKE) eunit' didn't run the tests in t_tests twice, etc."
	$t printf "%s\n" t t_tests x_tests | cmp app1/test-eunit.log -
	$t rm app1/test-eunit.log
	$i "Checking that '$(MAKE) eunit' returns non-zero for a failing test."
	$t rm -f app1/eunit/*
	$t printf "%s\n" \
		"-module(t_tests)." \
		'-include_lib("eunit/include/eunit.hrl").' \
		"succ_test() ->" \
		"	?assertEqual(42, t:succ(1))." \
		> app1/eunit/t_tests.erl
	$t $(MAKE) -C app1 distclean TEST_DIR=eunit $v
	$t if $(MAKE) -C app1 eunit TEST_DIR=eunit $v ; then false ; fi
	$t rm -rf app1/eunit app1/src/t.erl app1/test-eunit.log
	$i "Test 'eunit' passed."

# TODO: do coverage for 'tests' instead of 'eunit ct' when triq is fixed
tests-cover: app1
	$i "tests-cover: Testing 'eunit' and 'ct' with COVER=1"
	$i "Setting up eunit and ct suites."
	$t $(call create-module-t)
	$t mkdir -p app1/test
	$t printf "%s\n" \
		"-module(m_SUITE)." \
		"-export([all/0, testcase1/1])." \
		"all() -> [testcase1]." \
		"testcase1(_) -> 2 = m:succ(1)." \
	 > app1/test/m_SUITE.erl
	$i "Running tests with coverage analysis."
	$t $(MAKE) -C app1 eunit ct COVER=1 $v
	$t [ -e app1/test-eunit.log ]
	$t [ -e app1/eunit.coverdata ]
	$t [ -e app1/ct.coverdata ]
	$i "Generating coverage report."
	$t $(MAKE) -C app1 cover-report COVER=1 $v
	$t [ -e app1/cover/m.COVER.html ]
	$t [ -e app1/cover/t.COVER.html ]
	$t [ -e app1/cover/index.html ]
	$i "Checking combined coverage from eunit and ct."
	$t [ `grep 'Total: 100%' app1/cover/index.html | wc -l` -eq 1 ]
	$i "Checking that cover-report-clean removes cover report."
	$t $(MAKE) -C app1 cover-report-clean $v
	$t [ ! -e app1/cover ]
	$i "Checking that coverdata-clean removes cover data."
	$t $(MAKE) -C app1 coverdata-clean $v
	$t [ ! -e app1/eunit.coverdata ]
	@# clean up
	$t rm -rf app1/src/t.erl app1/test app1/test-eunit.log
	$t $(MAKE) -C app1 clean $v
	$i "Test 'tests-cover' passed."

docs: app1
	$i "docs: Testing EDoc including DOC_DEPS."
	$t printf "%s\n" \
		"PROJECT = app1" \
		"DOC_DEPS = edown" \
		"dep_edown = git https://github.com/uwiger/edown.git 0.7" \
		"EDOC_OPTS = {doclet, edown_doclet}" \
		"include erlang.mk" \
		"distclean:: distclean-doc-md" \
		"distclean-doc-md:" \
		"	rm -rf doc/*.md" \
		> app1/Makefile-doc
	$i "Downloading doc deps (edown) and building docs."
	$t $(MAKE) -C app1 -f Makefile-doc docs $v
	$i "Checking that '$(MAKE) docs' using edown generated a markdown file."
	$t [ -e app1/doc/m.md ]
	$i "Checking that '$(MAKE) distclean' deletes all generated doc files."
	$t $(MAKE) -C app1 -f Makefile-doc distclean $v
	$t [ "`ls app1/doc/`" = "" ]
	$i "Cleaning up test data."
	$t rm app1/Makefile-doc
	$i "Test 'docs' passed."

define app1_setup
	$i "Setting up app."
	$t mkdir -p app1
	$t cd .. && $(MAKE)
	$t cp ../erlang.mk app1/
	$t $(MAKE) -C app1 -f erlang.mk bootstrap-lib
	$t printf "%s\n" \
		"-module(m)." \
		"-export([succ/1])." \
		"succ(N) -> N + 1." \
		> app1/src/m.erl
endef
