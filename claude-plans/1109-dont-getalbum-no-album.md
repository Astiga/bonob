We're seeing HTTP 500 errors and users are experiencing problems when browsing playlists in Sonos via the Astiga content service.

The underlying data in Astiga (../Astiga/server.sql) is:

    mysql> select p.playlist_id, p.name, pe.name, pe.path from playlist p join playlist_entries pe on p.playlist_id=pe.playlis
    t_id  where p.id=38886;
    +-------------+-------------------------------+------------+-----------------------------------------------------------+
    | playlist_id | name                          | name       | path                                                      |
    +-------------+-------------------------------+------------+-----------------------------------------------------------+
    |        9559 | Arun O’Connor’s new tunes     | My Sropbox | /A.I Imagined Unreleased Songs/Do They.mp3                |
    |        9559 | Arun O’Connor’s new tunes     | My Sropbox | /A.I Imagined Unreleased Songs/Looking Back.mp3           |
    |        9559 | Arun O’Connor’s new tunes     | My Sropbox | /A.I Imagined Unreleased Songs/Rewind.mp3                 |
    |        9559 | Arun O’Connor’s new tunes     | My Sropbox | /A.I Imagined Unreleased Songs/Road To I Don’t Know.mp3   |
    |        9559 | Arun O’Connor’s new tunes     | My Sropbox | /A.I Imagined Unreleased Songs/Secondhand Heart.mp3       |
    |        9559 | Arun O’Connor’s new tunes     | My Sropbox | /A.I Imagined Unreleased Songs/The Landlord.mp3           |
    |        9559 | Arun O’Connor’s new tunes     | My Sropbox | /A.I Imagined Unreleased Songs/Younger Days.mp3           |
    +-------------+-------------------------------+------------+-----------------------------------------------------------+
    7 rows in set (0.00 sec)

The library:

    mysql> select l.name, l.path, l.album, l.artist from library l join playlist_entries pe where l.id=38886 and pe.name=l.nam
    e and pe.path=l.path;
    +------------+-----------------------------------------------------------+-------+--------+
    | name       | path                                                      | album | artist |
    +------------+-----------------------------------------------------------+-------+--------+
    | My Sropbox | /A.I Imagined Unreleased Songs/Do They.mp3                |       |        |
    | My Sropbox | /A.I Imagined Unreleased Songs/Looking Back.mp3           |       |        |
    | My Sropbox | /A.I Imagined Unreleased Songs/Rewind.mp3                 |       |        |
    | My Sropbox | /A.I Imagined Unreleased Songs/Road To I Don’t Know.mp3   |       |        |
    | My Sropbox | /A.I Imagined Unreleased Songs/Secondhand Heart.mp3       |       |        |
    | My Sropbox | /A.I Imagined Unreleased Songs/The Landlord.mp3           |       |        |
    | My Sropbox | /A.I Imagined Unreleased Songs/Younger Days.mp3           |       |        |
    +------------+-----------------------------------------------------------+-------+--------+
    7 rows in set (9.54 sec)

The user explored getPlaylists (see ../Astiga/play/routes.php for routes):

    GET /rest/getPlaylists

    <playlists>
        <playlist id="1" owner="gweinst@hotmail.com" name="Arun O’Connor’s new tunes" public="false" songCount="7" coverArt="2" created="2025-11-27T10:59:24"/>
    </playlists>

    GET /rest/getPlaylist?id=1

    <playlist id="1" name="Arun O’Connor’s new tunes" owner="gweinst@hotmail.com" songCount="7" duration="1579" created="2025-11-27T10:53:56">
        <entry id="3" title="Do They" genre="" size="1" contentType="audio/mpeg" suffix="mp3" duration="232" path="A.I Imagined Unreleased Songs/Do They.mp3" type="music" isDir="false" bitRate="180" created="2025-11-27T10:53:56"/>
        <entry id="4" title="Looking Back" genre="" size="1" contentType="audio/mpeg" suffix="mp3" duration="219" path="A.I Imagined Unreleased Songs/Looking Back.mp3" type="music" isDir="false" bitRate="181" created="2025-11-27T10:53:56"/>
        <entry id="5" title="Rewind" genre="" size="1" contentType="audio/mpeg" suffix="mp3" duration="236" path="A.I Imagined Unreleased Songs/Rewind.mp3" type="music" isDir="false" bitRate="189" created="2025-11-27T10:53:56"/>
        <entry id="6" title="Road To I Don’t Know" genre="" size="1" contentType="audio/mpeg" suffix="mp3" duration="210" path="A.I Imagined Unreleased Songs/Road To I Don’t Know.mp3" type="music" isDir="false" bitRate="181" created="2025-11-27T10:53:56"/>
        <entry id="7" title="Secondhand Heart" genre="" size="1" contentType="audio/mpeg" suffix="mp3" duration="209" path="A.I Imagined Unreleased Songs/Secondhand Heart.mp3" type="music" isDir="false" bitRate="179" created="2025-11-27T10:53:56"/>
        <entry id="8" title="The Landlord" genre="" size="1" contentType="audio/mpeg" suffix="mp3" duration="254" path="A.I Imagined Unreleased Songs/The Landlord.mp3" type="music" isDir="false" bitRate="179" created="2025-11-27T10:53:56"/>
        <entry id="9" title="Younger Days" genre="" size="1" contentType="audio/mpeg" suffix="mp3" duration="219" path="A.I Imagined Unreleased Songs/Younger Days.mp3" type="music" isDir="false" bitRate="187" created="2025-11-27T10:53:56"/>
    </playlist>

Note there are no `album` attributes.

We then see:

    rest/getAlbum

With no `id` attribute. This causes a HTTP 500. In the logs:

    [ERROR] Subsonic request failed: {
      path: '/rest/getAlbum',
      query: { id: undefined },
      rawResponse: undefined,
      error: 'Subsonic failed with: AxiosError: Request failed with status code 500',
      stack: undefined
    }
    {"level":"debug","message":{"data":"<?xml version=\"1.0\" encoding=\"utf-8\"?><soap:Envelope xmlns:soap=\"http://schemas.xmlsoap.org/soap/envelope/\"  xmlns:tns=\"http://www.sonos.com/Services/1.1\"><soap:Body><soap:Fault><soap:Code><soap:Value>SOAP-ENV:Server</soap:Value><soap:Subcode><soap:Value>InternalServerError</soap:Value></soap:Subcode></soap:Code><soap:Reason><soap:Text>Subsonic failed with: AxiosError: Request failed with status code 500</soap:Text></soap:Reason></soap:Fault></soap:Body></soap:Envelope>","level":"debug"},"service":"bonob","timestamp":"2025-11-27 07:21:41"}
    "POST /ws/sonos HTTP/1.1" 500 - "-" "Linux UPnP/1.0 Sonos/57.22-67250 (ICRU_iPhone12,1)"

If I make a call to rest/getAlbum with no id, I find it returns a HTTP 200 as expected.

### Tasks
- Why is a HTTP 500 returned inside bonob when calling our Subsonic endpoint?
- How do we avoid HTTP 500s and return an appropriate error code (`10: "The required param: id is missing"`) - see Subsonic API
- We should not call for albums if there's no album ID
 - Same for all downstream calls, e.g. artists