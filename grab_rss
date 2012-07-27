#!/usr/bin/python

__copyright__ = 'Copyright 2010, 2012 Jack Lloyd'
__license__   = 'GPLv2'
__author__    = 'Jack Lloyd'
__version__   = '0.3'
__email__     = 'lloyd@randombit.net'
__url__       = 'https://github.com/randombit/grab-rss'

"""
Dependencies:
  feedparser      - http://www.feedparser.org
  stripogram      - http://www.zope.org/Members/chrisw/StripOGram

Optional:
  dateutil.parser - http://labix.org/python-dateutil
  multiprocessing - http://code.google.com/p/python-multiprocessing/
"""

import feedparser
import stripogram

import logging
import optparse
import os
import re
import smtplib
import socket
import sqlite3
import sys
import time
import unicodedata
import ConfigParser

from email.MIMEText import MIMEText

try:
    import dateutil.parser

    def parse_date(date_string):
        dt = dateutil.parser.parse(date_string)
        return dt.strftime('%Y-%m-%d %H:%M')
except ImportError:
    def parse_date(date_string): return date_string # leave it unchanged

def conf_dir():
    if os.getenv('GRAB_RSS_DIR'):
        return os.getenv('GRAB_RSS_DIR')
    return os.path.join(os.getenv('HOME'), '.grab_rss')

def feedlist():
    feed_file = os.path.join(conf_dir(), 'feeds.txt')

    try:
        feeds = [s.strip() for s in open(feed_file).readlines() if not s.startswith('#')]
        logging.info('Read %d feeds from %s' % (len(feeds), feed_file))
        return feeds
    except IOError:
        raise Exception('No feeds found in %s' % (feed_file))

def read_config():
    config = ConfigParser.RawConfigParser(
        {
            'from': 'grab-rss@localhost',
            'smtp_host': 'localhost',
            'socket_timeout': '30',
            'user_agent': 'grab_rss ' + __version__ + ' ' + __url__,
            'pool_size': 0
        })

    config_file = os.path.join(conf_dir(), 'grab_rss.conf')

    try:
        config.readfp(open(config_file))
    except IOError:
        pass

    return config

class seen_items:
    def __init__(self, filename):
        self.db = sqlite3.connect(filename)
        self.cursor = self.db.cursor()
        self.cursor.execute('create table if not exists seen (timestamp integer, url text)')

    def save(self):
        self.db.commit()

    def remove_older_than(self, days):
        long_ago = int(time.time()) - int(days)*24*60

        self.cursor.execute('delete from seen where timestamp <= ?', [long_ago])
        return self.cursor.rowcount

    def seen_this_before(self, item):
        self.cursor.execute('select * from seen where url = ?', [item])
        there = self.cursor.fetchall()
        return len(there) > 0

    def note_as_seen(self, item):
        now = int(time.time())
        self.cursor.execute('insert into seen values(?, ?)', [now, item])

def wrap(text, width):
    # From http://code.activestate.com/recipes/148061/
    return reduce(lambda line, word, width=width: '%s%s%s' %
                  (line,
                   ' \n'[(len(line)-line.rfind('\n')-1
                         + len(word.split('\n',1)[0]
                              ) >= width)],
                   word),
                  text.split(' ')
                 )

def force_to_ascii(s):
    def replace_char(matches):
        u = matches.group(1)
        try:
            return unichr(int(u))
        except:
            return u

    s = re.sub("&#(\d+)(;|(?=\s))", replace_char, s)

    return unicodedata.normalize('NFKD', unicode(s)).encode('ascii', 'ignore')

def body_for(entry):
    timestamp = parse_date(entry.get('date', ''))

    link = force_to_ascii(entry.link)

    descr = wrap(
        stripogram.html2safehtml(
            force_to_ascii(entry.get('description', '')),
            valid_tags=('a')),
        80)

    return '\n\n'.join([timestamp, link, descr])

def parse_feed(url):
    logging.info('Reading %s' % (url))

    def feed_name(feed, feed_url):
        try:
            return feed['feed']['title']
        except:
            return feed_url.split('/')[3]

    # Don't return feed directly, PyCObjects are unpicklable and this
    # makes multiprocessing very unhappy
    feed = feedparser.parse(url)

    name = feed_name(feed, url)

    return (url, name, feed.entries)

def pool_map(pool_size, func, args):
    if pool_size > 0:
        import multiprocessing
        pool = multiprocessing.Pool(pool_size)
        return pool.map(func, args)
    else:
        return map(func, args)

def grab_feeds(state, pool_size):

    all_feeds = pool_map(pool_size, parse_feed, feedlist())

    for (feed_url, feed_title, feed) in all_feeds:
        num_entries = len(feed)

        if num_entries:
            logging.debug('Found %d entries in %s' % (num_entries, feed_url))
        else:
            logging.info('Found no entries in %s' % (feed_url))

        new_entries = 0

        for entry in feed:
            link = entry['link']

            if state.seen_this_before(link):
                continue

            title = entry.get('title', link)

            msg = MIMEText(body_for(entry))
            msg['X-GrabRSS-Feed'] = feed_url
            msg['Subject'] = force_to_ascii('%s - %s' % (feed_title, title))
            yield msg

            new_entries += 1

            state.note_as_seen(link)

        logging.debug('Saw %d new items from %s' % (new_entries, feed_url))

def main(argv = None):
    if argv is None:
        argv = sys.argv

    parser = optparse.OptionParser(version='%%prog %s' % (__version__))

    parser.add_option('-v', '--verbose', dest='verbose',
                      action='store_true', default=False,
                      help='be loud')

    parser.add_option('-q', '--quiet', dest='quiet',
                      action='store_true', default=False,
                      help='be quiet')

    parser.add_option('--dont-send', action='store_true', default=False,
                      help="don't actually send email")

    parser.add_option('--remove-older-than', dest='remove_older_than',
                      metavar='N',
                      help='remove seen.db entries older than N days')

    parser.add_option('--pool-size', metavar='N',
                      help='use N processes for pulling down feeds in parallel')

    (options, args) = parser.parse_args(argv)

    def log_level():
        if options.quiet: # -q overrides -v
            return logging.WARNING
        if options.verbose:
            return logging.DEBUG
        return logging.INFO

    logging.basicConfig(stream = sys.stdout,
                        format = '%(levelname) 7s: %(message)s',
                        level = log_level())

    config = read_config()

    socket.setdefaulttimeout(config.getint('GrabRSS', 'socket_timeout'))
    feedparser.USER_AGENT = config.get('GrabRSS', 'user_agent')

    pool_size = int(options.pool_size or config.get('GrabRSS', 'pool_size'))

    smtp_host = config.get('GrabRSS', 'smtp_host')

    logging.debug('Sending mail to %s' % (smtp_host))

    state = seen_items(os.path.join(conf_dir(), 'seen.db'))

    if options.remove_older_than:
        removed = state.remove_older_than(options.remove_older_than)
        logging.info('Removed %d old items' % (removed))
        state.save()
        return

    smtp = smtplib.SMTP(smtp_host)

    for email in grab_feeds(state, pool_size):
        if not options.dont_send:
            email['From'] = config.get('GrabRSS', 'from')
            email['To'] = config.get('GrabRSS', 'to')
            smtp.sendmail(email['From'], [email['To']], email.as_string())

    state.save()

    smtp.quit()

if __name__ == '__main__':
    sys.exit(main())
