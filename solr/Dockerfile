FROM solr:8.4.1
MAINTAINER  Alvaro Lopez "alopezme@redhat.com"

USER root

LABEL io.k8s.description="Run SOLR search in OpenShift" \
      io.k8s.display-name="SOLR 8.4.1" \
      io.openshift.expose-services="8983:http" \
      io.openshift.tags="solr,solr8.4.1"

# Change Solr directories ownership to root group (not root user)
# https://docs.openshift.com/container-platform/4.3/openshift_images/create-images.html#use-uid_create-images
RUN chgrp -R 0 /opt/solr \
  && chmod -R g=u /opt/solr

RUN chgrp -R 0 /opt/solr \
  && chmod -R g=u /opt/docker-solr

USER 8983
