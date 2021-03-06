.. image:: https://img.shields.io/travis/elemental-lf/benji/master.svg?style=plastic&label=Travis%20CI
    :target: https://travis-ci.org/elemental-lf/benji

.. image:: https://img.shields.io/pypi/l/benji.svg?style=plastic&label=License
    :target: https://pypi.org/project/benji/

.. image:: https://img.shields.io/pypi/v/benji.svg?style=plastic&label=PyPI%20version
    :target: https://pypi.org/project/benji/

.. image:: https://img.shields.io/pypi/pyversions/benji.svg?style=plastic&label=Supported%20Python%20versions
    :target: https://pypi.org/project/benji/

Benji Backup
============

Benji Backup is a block based deduplicating  backup software. It builds on the
excellent foundations and concepts of `backy² <http://backy2.com/>`_ by Daniel Kraft.
Many thanks go to him for making his work public and releasing backy² as
open-source software!

The primary use cases for Benji are:

* Fast and resource-efficient backup of Ceph RBD images to object or file storage
* Backup of LVM volumes (e.g. from servers or personal computers) to external hard
  drives or the cloud

Benji features a Docker image and Helm chart for integration with
`Kubernetes <https://kubernetes.io/>`_. This makes it easy to setup a backup solution 
for your persistent volumes.

Status
------


Benji is currently somewhere between alpha and beta quality I think. It passes all
included tests. The documentation isn't completely up-to-date. Please open an
issue on GitHub if you have a usage question that is not or incorrectly covered
by the documentation.

Benji requires **Python 3.6.5 or newer** because older Python versions
have some shortcomings in the ``concurrent.futures`` implementation which lead to an
excessive memory usage.

Older versions contained a Docker image for integrating with `Rook <https://rook.io/>`_.
As I no longer have access to a Rook installation and Rook changed its Docker base
image in the meantime I've dropped this support for the time being. The new generic
Kubernetes image (``benji-k8s``) can be used instead, but it will require some work to get
the Ceph credentials into the container. I'd accept patches for a third Docker
image (resurrecting the old ``benji-rook`` image) or maybe it's also possible to integrate
the changes into the ``benji-k8s`` image without too much fuss.

Upgrade notes
-------------

* Current master drops the version_statistics table and integrates the
  statistics into the versions table.  Statistics for still existing
  versions are moved into the versions table, but statistics for no
  longer existing versions are dropped! If you want to keep this data,
  you have to export it with ``benji stats`` or ``benji -m stats``
  before upgrading. Old metadata backups and exports can still be
  imported but the statistics in the versions table will be empty.

* The current master branch is not compatible with the old master branch of 10/05/2018
  and earlier and there is no migration path. The last commit supporting the old data
  structures and configuration file format is available under the tag master-20181005
  if you need it.

Main Features
-------------

**Small backups**
    Benji deduplicates while reading from the block device and only writes
    blocks once if they have the same checksum. Deduplication takes into
    account all historic data present in the backup storage target and so
    spans all backups and all backup sources. This can make deduplication
    more effective if images are clones of a common ancestor.

**Fast backups**
    With the help of Ceph's ``rbd diff``, Benji will only read the blocks
    that have changed since the last backup. Even when this information
    is not available (like with LVM) Benji will still only backup
    changed blocks.

**Fast restores**
    With supporting block storage (like Ceph's RBD), a sparse restore is
    possible. This means, sparse blocks (i.e. blocks which are holes or are
    all zeros) will be skipped on restore.

**NBD server facilitating file-based restores**
    Benji brings its own NBD (network block device) server which makes backup
    images directly mountable - even over the network on another machine. This
    enables file-based restores without restoring the whole image.

    These mounts are read/write (unless you specify ``-r``) and writing to them
    creates a copy-on-write backup version (**i.e. the original version is not modified**).
    This makes it possible to do repairs on the image (``fsck``, etc.) and restore
    the repaired copy afterwards.

**Small bandwidth requirements**
    As only changed blocks are written to the backup storage, a small connection
    is sufficient even for larger backups. Even with newly created block devices
    the traffic to the backup target is small, because these block devices usually
    contain mostly zeros and are deduplicated before reaching the target storage.

    In addition to this Benji supports fast state-of-the-art compression based on
    `zstandard <https://github.com/facebook/zstd>`_. This further reduces the
    required bandwidth and also reduces the storage space requirements.

**Support for a variety of backup storage targets**
    Benji supports AWS S3 as a data backend but also has options to enable
    compatibility with other S3 implementations like Google Storage, Ceph's
    RADOS Gateway or `Minio <https://www.minio.io/>`_.

    Benji also supports `Backblaze's <https://www.backblaze.com/>`_ B2 Cloud
    Storage which opens up a very cost effective way to keep your backups.

    Last but not least Benji can also use any file based storage including
    external hard drives and NFS based storage solutions.

**Confidentiality**
    Benji supports AES-256 in GCM mode to encrypt all your data on the backup
    storage. By using envelope encryption every block is encrypted with its
    own unique random key which makes plaintext attacks even more difficult.

**Integrity**
    Every backed up block keeps a checksum with it. When Benji scrubs the
    backup, it reads the block from the backup storage, calculates its
    checksum and compares it to the stored checksum. If the checksum differs,
    it's most likely that there was an error while storing or reading
    the block, or because of bit rot on the backup target storage.

    Benji also supports a faster light-weight scrubbing mode which only checks
    the object's existence and metadata consistency.

    If a scrubbing failure occurs, the defective block and the backups it belongs
    to are marked as 'invalid' and the block will be re-read for the next backup
    version even if ``rbd diff`` indicates that it hasn't changed.

    Scrubbing can also take a percentage value of how many blocks of the backup
    it should scrub. So you can statistically scrub 16% each day and have a
    full scrub each week (16*7 > 100).

**Concurrency: Backup while scrubbing while restoring**
    As Benji is a long-running process, you don't want to wait until something has
    finished of course. You can scrub, backup and restore at the same time and
    multiple times each.

    Benji even supports distributed operation where multiple instances run on
    different hosts or in different containers at the same time.

**Cache friendly**
    While reading large pieces of data on Linux, buffers and caches get filled
    up with data, which in case of backups is essentially only needed once.
    Benji instructs Linux and Ceph to immediately forget the data once it's processed.

**Simplicity: As simple as cp, but as clever as a backup solution needs to be**
    With a small set of commands, good ``--help`` and intuitive usage,
    Benji feels mostly like ``cp``. And that's intentional, because we think,
    a restore must be fool-proof and succeed even if you're woken up at 3am in the
    morning.

**Prevents you from doing something stupid**
    By providing a configuration value for how old backups need to be in order to
    be able to remove them, you can't accidentally remove very young backups. An
    exception to this is the enforcement of retention policies which will also
    remove recent backups if configured.

    With ``benji protect`` you can protect versions from being removed.
    This is important when you plan to restore a version which according to the
    retention policy may be removed soon. During restore a lock will also prevent
    removal, however, by protecting it, it cannot be removed until you decide
    that it is no longer needed.

    Also, you'll need to use ``--force`` to overwrite existing files or volumes.

**Free and Open Source Software**
    Anyone can review the source code and audit security and functionality.
    Benji is licensed under the LGPLv3 license. Please see the documentation
    for a full list of licenses.
