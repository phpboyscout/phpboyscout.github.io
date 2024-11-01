---
title: "Migrating away from Mediawiki and how to export its data"
date: "2020-08-19"
categories: 
  - "development"
---

I like Mediawiki, it is a simple tool capable of doing a lot and can be very flexible and easy to customise. However its not always the right solution! I had a situation where we needed to migrate away from using it for a combination of security and usability reasons. So I thought it would be good to document it.

After reviewing a few things it was decided to move things over to the companies already existing O365 SharePoint as a new site. This sounded laborious as first, but actually turned out to be pretty straight forward.

We start with getting data out of Mediawiki, thankfully we only wanted the most recent revision and not the full history of a page. We use PostgreSQL as a back-end so it was reasonably straight forward to figure out how to extract the data in a sensible query.

```sql
SELECT 
  page_id as id, 
  page.page_title as title, 
  pagecontent.old_text as content, 
  page_touched as edited
FROM mediawiki.page
LEFT JOIN mediawiki.slots ON page.page_latest = slots.slot_revision_id
LEFT JOIN mediawiki.content ON content.content_id = slots.slot_content_id
LEFT JOIN mediawiki.pagecontent ON pagecontent.old_id = CAST(OVERLAY(content.content_address placing '' from 1 for 3) as integer)
ORDER BY page_touched DESC;
```

It tool a little sleuthing to realize that the `slots` table was the pivotal in extracting the latest page version. With the right join and a little mangling of the `content_address` field from the `contents` table to remove the "tt:" from the value and convert to an integer we now have a result set of all the page names and the latest revision of that page. I also added in the date the page was last updated to allow me to see when it was last edited as it was a live system migration and helped me to ensure content remained sync while both were still in play.

Once I had the query it was a simple job of writing an "Exporter" using Go Lang to extract the data and write it to files, I'll chuck a snippet of code at the bottom of the post for you.

Mediawiki uses `wikitext` as a format so I needed to convert it to something more widely understood. Having used Pandoc in the past successfully I plumped for this as I knew it would handle a lot of options and was simple to use to convert to the `markdown_mmd` format

I Installed it via the ubuntu apt package available on my system and wired this in as a hacky `exec` command into my script... and voila! I had hardcopies of all the Mediawiki pages on my system in both `wikitext` and `markdown_mmd` format.

Why `markdown_mmd` I hear you ask... mainly because it gave me the cleanest conversion for use with the new markdown web page widget for Sharepoint's modern interface.

Now we have the files we could do a little munging and parsing to convert URLs into the format needed for the new location in Sharepoint, easily done with a bit of regex pattern matching, which I wont go into as yours will be very different from mine... suffice to say looking for `"wikilink"` in my regex helped massively in finding all the occurrences I needed to update. I used `sed` but you could use whatever tool you like or add it into your version of the exporter

```
'SysAdmin/(.+) "wikilink"' 
```

and with a little back referencing to substitute the values we need to keep and its all good.

Next came the import of the data into Sharepoint, but that is a post for another day.

```
package data

import (
	"bytes"
	"fmt"
	"github.com/jmoiron/sqlx"
	"github.com/rs/zerolog/log"
	"io/ioutil"
	"os"
	"os/exec"
	"path/filepath"
	"time"
	"wiki-export/src/util"
)

type Page struct {
	Id int
	Title    string
	Content string
	Edited time.Time
}

type Exporter struct {
	Config util.ExporterConfig
	DB     *sqlx.DB
}

func (l *Exporter) Export() {
	stmt := `
		SELECT page_id as id, page.page_title as title, pagecontent.old_text as content, page_touched as edited
		FROM mediawiki.page
		LEFT JOIN mediawiki.slots ON page.page_latest = slots.slot_revision_id
		LEFT JOIN mediawiki.content ON content.content_id = slots.slot_content_id
		LEFT JOIN mediawiki.pagecontent ON pagecontent.old_id = CAST(OVERLAY(content.content_address placing '' from 1 for 3) as integer)
		ORDER BY page_touched DESC
		;`

	page := Page{}
	rows, err := l.DB.Queryx(stmt)
	util.CheckErr(err)

	for rows.Next() {
		util.CheckErr(rows.StructScan(&page))
		wikiFilename := fmt.Sprintf("%s.mediawiki",filepath.Base(page.Title))
		mdFilename := fmt.Sprintf("%s.md",filepath.Base(page.Title))
		path := filepath.Dir(page.Title)
		wikiDir := fmt.Sprintf("%s/mediawiki",l.Config.TargetDir)
		mdDir := fmt.Sprintf("%s/%s",l.Config.TargetDir, l.Config.TargetFormat)

		if path != "." {
			wikiDir = fmt.Sprintf("%s/mediawiki/%s",l.Config.TargetDir , path)
			mdDir = fmt.Sprintf("%s/md/%s",l.Config.TargetDir , path)
		}
		util.CheckErr(os.MkdirAll(wikiDir, 0777))
		util.CheckErr(os.MkdirAll(mdDir, 0777))

		wikiTarget := fmt.Sprintf("%s/%s", wikiDir, wikiFilename)
		mdTarget := fmt.Sprintf("%s/%s", mdDir, mdFilename)

		log.Debug().Msgf("%s => %s -> %s", page.Title, wikiTarget, mdTarget)

		c := []byte(page.Content)
		util.CheckErr(ioutil.WriteFile(wikiTarget, c, 0777))

		cmd := exec.Command("pandoc", "-f","mediawiki", "-t", l.Config.TargetFormat, wikiTarget)
		
		var errorBuffer bytes.Buffer
		var outputBuffer bytes.Buffer
		cmd.Stdout = &outputBuffer
		cmd.Stderr = &errorBuffer
		err := cmd.Run()
		if err != nil {
			log.Err(err).Msgf("ERROR: %s", errorBuffer.String())
			util.CheckErr(err)
		}

		util.CheckErr(ioutil.WriteFile(mdTarget, outputBuffer.Bytes(), 0777))
	}
}
```
