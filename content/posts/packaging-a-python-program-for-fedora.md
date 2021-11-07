---
title: "Packaging a Python program for Fedora"
date: 2019-09-29T13:14:02+03:00
showDate: true
draft: false
---

Being a Fedora contributor is one of my long term goals. Recently I took the
first step by submitting the [Redmine
CLI](https://github.com/egegunes/redmine-cli) to Fedora. Creating spec file is
a bit hard, even for a trivial command line application. So I'll document the
steps others to benefit.

## Spec file

```
%global pypi_name redminecli

%{?python_enable_dependency_generator}

Name:           %{pypi_name}
Version:        1.1.8
Release:        1%{?dist}
Summary:        Command line interface for Redmine

License:        GPLv3
URL:            https://github.com/egegunes/redmine-cli
Source0:        %{pypi_source}

BuildArch:      noarch
BuildRequires:  python3-devel
BuildRequires:  python3-setuptools
BuildRequires:  python3-pytest
BuildRequires:  python3-click
BuildRequires:  python3-requests

%description
`redminecli` is a command line interface for Redmine.

%prep
%autosetup -n %{pypi_name}-%{version}

%build
%py3_build

%install
%py3_install

%files
%doc README.md
%{_bindir}/redmine
%{python3_sitelib}/redmine/
%{python3_sitelib}/%{pypi_name}-*.egg-info/

%check
%{__python3} -m pytest

%changelog
* Thu Sep 26 2019 Ege Güneş <egegunes@gmail.com> - 1.1.8-1
- Bump to 1.1.8

* Thu Sep 26 2019 Ege Güneş <egegunes@gmail.com> - 1.1.7-1
- Bump to 1.1.7

* Tue Aug 27 2019 Ege Güneş <egegunes@gmail.com> - 1.1.6-1
- Bump to 1.1.6

* Sun Aug 25 2019 Ege Güneş <egegunes@gmail.com> - 1.1.5-1
- Bump to 1.1.5

* Sun Aug 11 2019 Ege Güneş <egegunes@gmail.com> - 1.1.4-1
- Bump to 1.1.4

* Sat Aug 10 2019 Ege Güneş <egegunes@gmail.com>
- Initial package
```

Most of the spec is straight forward, but some settings may require
explanation:

`python_enable_dependency_generator`: In spec, you define two types of
dependencies: build dependency and runtime dependency. I defined build
dependencies with `BuildRequires` but there is no `Requires` as you can see.
This setting automatically generates runtime dependencies from package
metadata.

`BuildArch`: Package's targeted architecture (ie. x86_64). Since package is a
Python program it should be run on all.

`py3_build`: It's a smart macro for `python3 setup.py build`.

`py3_install`: It's a smart macro for `python3 setup.py install`.

`%files`: Files listed under this section is important. All files installed to
the system by package MUST be listed here. If your package installs a file not
listed build fails.

`%check`: If the package has tests this is the place to run them. Tests really
confused me. My reviewer demanded to run tests but tests depend on some runtime
dependencies and [Fedora Python Packaging
Guidelines](https://docs.fedoraproject.org/en-US/packaging-guidelines/Python/)
explicitly declares "Python modules must not download any dependencies during
the build process." But the reviewer said it's OK to add them as build
dependency for tests. Initially, I had only `python3-devel` and
`python3-setuptools` as `BuildRequires` but for tests I added `python3-pytest`,
`python3-requests` and `python3-click` too.

`%changelog`: This section is for documenting the changes to **spec** not the
upstream program.

After you create the spec, you'll create a source RPM from it. There is a [nice
post about source
RPM](https://fedoramagazine.org/how-rpm-packages-are-made-the-source-rpm/) on
Fedora Magazine.

First, you need to prepare your system:

```
$ dnf install fedora-packager
$ rpmdev-setuptree
$ tree rpmbuild/
rpmbuild/
├── BUILD
├── RPMS
├── SOURCES
├── SPECS
└── SRPMS

5 directories, 0 files
```

Then, download source from PyPI to `~/rpmbuild/SOURCES` and build source RPM:

```
$ rpmbuild -bs redminecli.spec
$ tree rpmbuild/
rpmbuild/
├── BUILD
├── BUILDROOT
├── RPMS
├── SOURCES
│   └── redminecli-1.1.8.tar.gz
├── SPECS
│   └── redminecli.spec
└── SRPMS
    └── redminecli-1.1.8-1.fc30.src.rpm

6 directories, 3 files
```

## FAS Account

You need to have a [FAS account](https://admin.fedoraproject.org/accounts/) to submit packages to Fedora.

## Koji

[Koji](https://koji.fedoraproject.org/koji/) is Fedora's RPM build system. You
can build packages against specific architectures and Fedora releases.

First, you need to get a Kerberos ticket:

```
$ KRB5_TRACE=/dev/stdout kinit your_fas_username@FEDORAPROJECT.ORG
```

Then, you can start a build from command line:

```
$ koji build --scratch f30 ~/rpmbuild/SRPMS/redminecli-1.1.8-1.fc30.src.rpm
```

You can see the build status on the web UI and if build fails you can check build logs for
errors.

## COPR

To submit a package review request, the spec and the source RPM have to
publicly accessible for reviewers. [COPR](https://copr.fedorainfracloud.org/)
is for building and creating third party RPM packages and repositories. You can
build your package on COPR and point reviewers to your repository.

## Bugzilla

After all these steps, you need to open a
[Bugzilla](https://bugzilla.redhat.com/) ticket and request a review for your
package. Add SRPM and Spec urls and latest **successful** Koji build url to
description.

Then, you'll wait for someone to review your package. You may need to make
changes on the spec if a reviewer demands it till someone from packager group
[approves](https://bugzilla.redhat.com/show_bug.cgi?id=1739816#c9) your package.

## Next steps

If it's your first package, after it's approved you need to find a sponsor to
join the packagers group. See
[documentation](https://fedoraproject.org/wiki/How_to_get_sponsored_into_the_packager_group) about sponsorship process.

Now, Redmine CLI is approved and I need to find a sponsor. This post is a first
step to find one. Then, I'm going to do some informal reviews to show I
understand packaging process and most of the best practices.
