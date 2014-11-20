---
layout: post   
title: Debug a vmware API migrate issue   
description: Use the vmware API to check the SN info, but the result is null, so let's debug with the scenes.  
category: operation
---

##Plot reproduce
When we use below snippet codes to gather SN info:   		

	my $client = WSMan::StubOps->new(%args);
    my @computersystem;
    eval {
                @computersystem = $client->EnumerateInstances(class_name => 'CIM_PhysicalPackage');
   		};
        if ($@) {
                 print "Check the error message: $@";
                return "";
        }
      

The output messages is:			
			
	Undefined subroutine &UUID::generate called at /usr/lib64/perl5/WSMan/WSBasic.pm

###How to solve it
I found the root cause is the UUID version. So we need update it:

	yum install libuuid-devel
	wget --directory-prefix=/tmp http://search.cpan.org/CPAN/authors/id/J/JN/JNH/UUID-0.04.tar.gz
	tar -zxvf /tmp/UUID-0.04.tar.gz -C /tmp
	cd /tmp/UUID-0.04
	perl Makefile.PL
	make
	make install	






      