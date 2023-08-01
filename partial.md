
##----------------------------------------------------------
##
## creare tabele client, cont, tip_operatiune, tranzactie_cont
##
##-------------------------------------------------------------


DROP TABLE IF EXISTS cont;
DROP TABLE IF EXISTS client;
CREATE TABLE IF NOT EXISTS client
(id int auto_increment primary key,
nume char(20),
prenume char(20),
adresa char(50));

CREATE TABLE IF NOT EXISTS cont
(id int auto_increment primary key,
client_id int,
cod int,
valoare float,
FOREIGN KEY (client_id) REFERENCES client(id)

);

CREATE TABLE IF NOT EXISTS tip_operatiune    
(id int auto_increment primary key,
nume char(40));

CREATE TABLE IF NOT EXISTS tranzactie_cont
(id int auto_increment primary key,
cont_sursa_id int,
cont_destinatie_id int,
tip int,
data date,
timp time(2),
valoare float,
detaliu char(100),
FOREIGN KEY (cont_sursa_id) REFERENCES cont(id),
FOREIGN KEY (cont_destinatie_id) REFERENCES cont(id),
FOREIGN KEY (tip) REFERENCES tip_operatiune(id));



alter table cont 
add stare char(7) not null default 'DESCHIS' ;


##----------------------------------------------------------
##
## creare triggere
##
##----------------------------------------------------------


DROP TRIGGER IF EXISTS insert_cont_client;

delimiter //

CREATE TRIGGER insert_cont_client AFTER INSERT INTO client
  FOR EACH ROW BEGIN
    INSERT INTO cont (client_id, cod, valoare) VALUES(NEW.id, NEW.id * 10001, 0);
  END; //

delimiter ;

## - Trigger AFTER INSERT tabela cont
DROP TRIGGER IF EXISTS adauga_tranzactie_creare_cont;

delimiter //

CREATE TRIGGER adauga_tranzactie_creare_cont AFTER INSERT INTO cont
  FOR EACH ROW BEGIN
    INSERT INTO  tranzactie_cont
    (cont_sursa_id, cont_destinatie_id,tip, data, timp, valoare, detaliu) VALUES
    (NEW.id, NEW.id,
    (SELECT id from tip_operatiune WHERE nume = 'Deschidere cont'),
     CURRENT_DATE(), NOW(2),0, 'Deschidere cont');
  END; //

delimiter ;


##----------------------------------------------------------
##
## inserare data in tabelele de client, tip_operatiune
##
##-------------------------------------------------------------
insert into tip_operatiune
(nume)
VALUES
('Deschidere cont'),
('Inchidere cont'),
('Transfer intre conturi'),
('Depunere ghiseu'),
('Depunere ATM'),
('Extragere ghiseu'),
('Extragere ATM'),
('Balanta cont'),
('Istoric tranzactii'),
('TR_ERR: cont creditor insuficient');

INSERT INTO client
(nume, prenume, adresa)
VALUES
('Popescu', 	'Maria',  'Blvd. Primaverii, Bucuresti'),
('Andreescu', 	'Victor', 'Blvd. Iuliu Maniu, Bucuresti'),
('Predescu', 	'Andreea', 'Str. Virtutii, Baia Mare'),
('Ionescu', 	'Mihaela', 'Str. Visinilor, Bucuresti'),
('Dinu', 	'Cornel', 'Calea Victoriei, Bucuresti'),
('Vodafone',    'S.A',   'Bucuresti'),
('Romtelecon',   'S.A',   'Bucuresti');



INSERT INTO client
(nume, prenume, adresa)
VALUES
('Popa', 	'Ion', 	  'Str. Mare, Dabuleni, Ilfov');


##----------------------------------------------------------
##
## datele sunt inserate automat in cont;
##
##-------------------------------------------------------------

## UPDATE cont SET  valoare = 2000 WHERE client_id = 1;
## UPDATE cont SET  valoare = 1000 WHERE client_id = 2;
## UPDATE cont SET  valoare = 2000 WHERE client_id = 3;
## UPDATE cont SET  valoare = 4500 WHERE client_id = 4;
## UPDATE cont SET  valoare = 6000 WHERE client_id = 5;
## UPDATE cont SET  valoare = 7000 WHERE client_id = 6;
## UPDATE cont SET  valoare = 100000 WHERE client_id = 7;
## UPDATE cont SET  valoare = 200000 WHERE client_id = 8;
## UPDATE cont SET  valoare = 2500 WHERE client_id = 9;



# Alimentare cont (ghiseu)
DROP PROCEDURE IF EXISTS ALIMENTARE_CONT;

DELIMITER //

CREATE PROCEDURE ALIMENTARE_CONT(cont_destinatie_cod int, valoare_depunere float)

  BEGIN
    START TRANSACTION;
       SET @ID_D = NULL, @VAL = NULL, @ID_TIP = NULL, @STARE_CONT = NULL;
       SELECT @ID_D := id FROM cont WHERE cod = cont_destinatie_cod;
       SELECT @VAL := valoare FROM cont WHERE id = @ID_D;
	SELECT	@STARE_CONT := stare FROM cont WHERE id = @ID_D;
       SELECT @ID_TIP := id FROM tip_operatiune WHERE nume = 'Depunere ghiseu';
	   
       IF(@ID_D IS NOT NULL) AND (@STARE_CONT = ‘deschis’) THEN
	 BEGIN
	   INSERT INTO tranzactie_cont
	    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
	    VALUES
	    (NULL, @ID_D, valoare_depunere, @ID_TIP, CURRENT_DATE(), NOW(2),'Depunere ghiseu');

	    UPDATE cont SET valoare = valoare + valoare_depunere WHERE id = @ID_D;

	   COMMIT;
         END;

	ELSE ROLLBACK;
    	END IF;
  END //
DELIMITER ;


# Transfer din cont in cont
DROP PROCEDURE IF EXISTS TRANSFER_INTRE_CONTURI;

DELIMITER //

CREATE PROCEDURE
   TRANSFER_INTRE_CONTURI(cont_sursa_cod int, cont_destinatie_cod int,
   			valoare_transfer float, detaliu_transfer char(20))

  BEGIN
    START TRANSACTION;
    	    SET @ID_S = NULL, @ID_D = NULL, @VAL = NULL, @ID_TIP1 = NULL, @ID_TIP2 = NULL;
    	    SELECT @ID_S := id FROM cont WHERE cod = cont_sursa_cod;
    	    SELECT @ID_D := id FROM cont WHERE cod = cont_destinatie_cod;
    	    SELECT @VAL := valoare FROM cont WHERE id = @ID_S;
    	    SELECT @ID_TIP1 := id FROM tip_operatiune WHERE nume = 'Transfer intre conturi';
    	    SELECT @ID_TIP2 := id FROM tip_operatiune WHERE nume = 'TR_ERR: cont creditor insuficient';

    	    IF (@VAL < valoare_transfer) THEN
		    BEGIN
			INSERT INTO tranzactie_cont
			    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
			    VALUES
			    (@ID_S, @ID_D, 0, @ID_TIP2, CURRENT_DATE(), NOW(2),detaliu_transfer);
		    	COMMIT;
		    END;
	    ELSEIF (@ID_S IS NOT NULL AND @ID_D IS NOT NULL) THEN
		    BEGIN
			INSERT INTO tranzactie_cont
			    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
			    VALUES
			    (@ID_S, @ID_D, valoare_transfer, @ID_TIP1, CURRENT_DATE(), NOW(2),detaliu_transfer);

			UPDATE cont SET valoare = valoare - valoare_transfer WHERE id = @ID_S;

   			UPDATE cont SET valoare = valoare + valoare_transfer WHERE id = @ID_D;

			COMMIT;
		    END;

	   ELSE ROLLBACK;
    	   END IF;
  END //
DELIMITER ;



# Extragere de la ATM
DROP PROCEDURE IF EXISTS EXTRAGERE_ATM;

DELIMITER //

CREATE PROCEDURE EXTRAGERE_ATM(cont_sursa_cod int, valoare_extragere float)

  BEGIN
    START TRANSACTION;
       	    SET @ID_S = NULL, @VAL = NULL, @ID_TIP1 = NULL, @ID_TIP2 = NULL;
    	    SELECT @ID_S := id FROM cont WHERE cod = cont_sursa_cod;
    	    SELECT @VAL := valoare FROM cont WHERE id = @ID_S;
    	    SELECT @ID_TIP1 := id FROM tip_operatiune WHERE nume = 'Extragere ATM';
    	    SELECT @ID_TIP2 := id FROM tip_operatiune WHERE nume = 'TR_ERR: cont creditor insuficient';

    	    IF (@ID_S IS NOT NULL AND @VAL < valoare_extragere) THEN
		    BEGIN
			INSERT INTO tranzactie_cont
			    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
			    VALUES
			    (@ID_S, NULL, 0, @ID_TIP2, CURRENT_DATE(), NOW(2),'Extragere ATM esuata');
		    	COMMIT;
		    END;
	    ELSEIF (@ID_S IS NOT NULL) THEN
		    BEGIN
			INSERT INTO tranzactie_cont
			    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
			    VALUES
			    (@ID_S, NULL, valoare_extragere, @ID_TIP1, CURRENT_DATE(), NOW(2),'Extragere ATM reusita');

			UPDATE cont SET valoare = valoare - valoare_extragere WHERE id = @ID_S;

			COMMIT;
		    END;

	   ELSE ROLLBACK;
    	   END IF;
  END //
DELIMITER ;

# Procedure istoric tranzactii
DROP PROCEDURE IF EXISTS ISTORIC_TRANZACTII;

DELIMITER //

CREATE PROCEDURE ISTORIC_TRANZACTII(cod_cont int, data_initiala date, data_finala date, stocheaza int)

  BEGIN
  	IF(stocheaza = 1) THEN
  	BEGIN

	    START TRANSACTION;

		SET @ID = NULL;
		SELECT @ID := id FROM cont WHERE cod = cod_cont;

		INSERT INTO tranzactie_cont
		(cont_sursa_id, tip, data, timp,detaliu)
		VALUES
		(@ID,(SELECT id from tip_operatiune WHERE nume = 'Istoric tranzactii'),
		CURRENT_DATE(), NOW(2),'Consultare tranzactii');
	    COMMIT;
	END;
	END IF;

	SELECT cont_sursa_id AS Sursa, cont_destinatie_id AS Destinatie,
	tranzactie_cont.valoare AS Valoare,  tip_operatiune.nume AS Operatiune, data AS Data, timp AS Timp,
	detaliu as Detaliu
	FROM tranzactie_cont, tip_operatiune, cont, client
	WHERE tranzactie_cont.tip = tip_operatiune.id
	AND ( cont.id = cont_sursa_id OR cont.id = cont_destinatie_id)
	AND  client.id = cont.id
	AND cont.cod = cod_cont
	AND data BETWEEN data_initiala AND  data_finala
	ORDER BY Data, Timp;

  END //
DELIMITER ;


# Balanta cont (la o data anume)
DROP PROCEDURE IF EXISTS BALANTA_CONT;

DELIMITER //

CREATE PROCEDURE BALANTA_CONT(cont_cod int, data_balanta date, ora_balanta time(2))

  BEGIN
    SET @ID = NULL, @ID_TIP = NULL;
    SELECT @ID := id FROM cont WHERE cod = cont_cod;
    SELECT @ID_TIP := id FROM tip_operatiune WHERE nume = 'Balanta cont';

    IF(@ID IS NOT NULL) THEN
    	BEGIN
		SELECT @Credit:= SUM(valoare)
		FROM tranzactie_cont
		WHERE (data < data_balanta OR (data = data_balanta AND timp <= ora_balanta))
		GROUP BY cont_destinatie_id
		HAVING tranzactie_cont.cont_destinatie_id = @ID;

		SELECT @Debit:= SUM(valoare)
		FROM tranzactie_cont
		WHERE (data < data_balanta OR (data = data_balanta AND timp <= ora_balanta))
		GROUP BY cont_sursa_id
		HAVING tranzactie_cont.cont_sursa_id = @ID;

		SELECT cont_cod AS Cont, @Credit - @Debit AS Balanta,
		data_balanta AS Data, ora_balanta As Ora;

	END;
   END IF;
  END //
DELIMITER ;









#####################################################################################################################################################



##----------------------------------------------------------
##
## Implementare proceduri stocate lipsa
##	-	ALIMENTARE_CONT_ATM
##	-	EXTRAGERE_GHISEU
##	-	INCHIDERE CONT
##-------------------------------------------------------------

# Alimentare cont (ATM)
DROP PROCEDURE IF EXISTS ALIMENTARE_CONT_ATM;

DELIMITER //

CREATE PROCEDURE ALIMENTARE_CONT_ATM(cont_destinatie_cod int, valoare_depunere float)

  BEGIN
    START TRANSACTION;
       SET @ID_D = NULL, @VAL = NULL, @ID_TIP = NULL;
       SELECT @ID_D := id FROM cont WHERE cod = cont_destinatie_cod;
       SELECT @VAL := valoare FROM cont WHERE id = @ID_D;
	SELECT @STARE := stare FROM cont WHERE id = @ID_D;
       SELECT @ID_TIP := id FROM tip_operatiune WHERE nume = 'Depunere ATM';

       IF(@ID_D IS NOT NULL AND @STARE = “deschis”) THEN
	 BEGIN
	   INSERT INTO tranzactie_cont
	    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
	    VALUES
	    (NULL, @ID_D, valoare_depunere, @ID_TIP, CURRENT_DATE(), NOW(2),'Depunere ATM');

	    UPDATE cont SET valoare = valoare + valoare_depunere WHERE id = @ID_D;

	   COMMIT;
         END;

	ELSE ROLLBACK;
    	END IF;
  END //
DELIMITER ;





# Extragere de la ghiseu
DROP PROCEDURE IF EXISTS EXTRAGERE_GHISEU;

DELIMITER //
CREATE PROCEDURE EXTRAGERE_GHISEU(cont_sursa_cod int, valoare_extragere float)

  BEGIN
    START TRANSACTION;
       	    SET @ID_S = NULL, @VAL = NULL, @ID_TIP1 = NULL, @ID_TIP2 = NULL;
    	    SELECT @ID_S := id FROM cont WHERE cod = cont_sursa_cod;
    	    SELECT @VAL := valoare FROM cont WHERE id = @ID_S;
    	    SELECT @ID_TIP1 := id FROM tip_operatiune WHERE nume = 'Extragere ghiseu';
    	    SELECT @ID_TIP2 := id FROM tip_operatiune WHERE nume = 'TR_ERR: cont creditor insuficient';
		SELECT @STARE := stare FROM cont WHERE id = @ID_D;

    	    IF (@ID_S IS NOT NULL AND (@VAL < valoare_extragere OR @STARE = “deschis”)) THEN
		    BEGIN
			INSERT INTO tranzactie_cont
			    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
			    VALUES
			    (@ID_S, NULL, 0, @ID_TIP2, CURRENT_DATE(), NOW(2),'Extragere ghiseu esuata');
		    	COMMIT;
		    END;
	    ELSEIF (@ID_S IS NOT NULL) THEN
		    BEGIN
			INSERT INTO tranzactie_cont
			    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
			    VALUES
			    (@ID_S, NULL, valoare_extragere, @ID_TIP1, CURRENT_DATE(), NOW(2),'Extragere reusita.');

			UPDATE cont SET valoare = valoare - valoare_extragere WHERE id = @ID_S;

			COMMIT;
		    END;

	   ELSE ROLLBACK;
    	   END IF;
  END //
DELIMITER ;


# Inchidere cont
DROP PROCEDURE IF EXISTS INCHIDERE_CONT;

DELIMITER //

CREATE PROCEDURE INCHIDERE_CONT(cont int)

  BEGIN
    START TRANSACTION;
       ##SET @ID_D = NULL, @ID_TIP = NULL, @ID_CLIENT = NULL, @NUME_CLIENT = NULL, @PRENUME_CLIENT = NULL;
	SELECT @STARE := stare FROM cont WHERE id = @ID_D;
       SELECT @ID_D := id, @ID_CLIENT:= client_id FROM cont WHERE cod = cont;
	SELECT @VAL := valoare FROM cont WHERE id = @ID_D;
       SELECT @ID_TIP := id FROM tip_operatiune WHERE nume = 'Inchidere cont';	    
	   
       IF(@ID_D IS NOT NULL AND (@VAL = 0 AND @STARE != “inchis”)) THEN
	 BEGIN
	   INSERT INTO tranzactie_cont
	    (cont_sursa_id, cont_destinatie_id, valoare, tip, data, timp,detaliu)
	    VALUES
	    (@ID_D, @ID_D, 0, @ID_TIP, CURRENT_DATE(), NOW(2), 'Contul inchis');

		UPDATE cont set stare = 'INCHIS' WHERE client_id=@ID_CLIENT;
		##UPDATE cont set valoare = 0 WHERE client_id=@ID_CLIENT;
		
	   COMMIT;
         END;

	ELSE ROLLBACK;
    	END IF;
  END //
DELIMITER ;



##-------------------------------------------------------------


##----------------------------------------------------------
##
## 		TESTARE
##-------------------------------------------------------------


CALL ALIMENTARE_CONT(10001,2000);
CALL ALIMENTARE_CONT(20002,1000);
CALL ALIMENTARE_CONT(30003,2000);
CALL ALIMENTARE_CONT(40004,4500);
CALL ALIMENTARE_CONT(50005,6000);
CALL ALIMENTARE_CONT(60006,7000);
CALL ALIMENTARE_CONT(70007,100000);
CALL ALIMENTARE_CONT(80008,200000);
CALL ALIMENTARE_CONT(90009,2500);

CALL TRANSFER_INTRE_CONTURI(10001,20002,3000,'Ramburs imprumut');


CALL ISTORIC_TRANZACTII(10001, CURRENT_DATE(), CURRENT_DATE(),1);
CALL TRANSFER_INTRE_CONTURI(10001,20002,100,'Ramburs imprumut');
CALL TRANSFER_INTRE_CONTURI(70007,10001,1000,'Plata salariu');
CALL TRANSFER_INTRE_CONTURI(10001,80008,120,'Plata utilitati');
CALL ISTORIC_TRANZACTII(10001, CURRENT_DATE(), CURRENT_DATE(),1);
CALL ISTORIC_TRANZACTII(10001, CURRENT_DATE(), CURRENT_DATE(),1);

CALL TRANSFER_INTRE_CONTURI(10001,40004,450,'Cumparaturi diverse');
CALL TRANSFER_INTRE_CONTURI(10001,40004,4500,'Plata imprumut');
CALL TRANSFER_INTRE_CONTURI(100001,40004,450,'Plata imprumut');
CALL ISTORIC_TRANZACTII(10001, CURRENT_DATE(), CURRENT_DATE(),1);

CALL ISTORIC_TRANZACTII(20002, CURRENT_DATE(), CURRENT_DATE(),0);
CALL TRANSFER_INTRE_CONTURI(20002,30003,200,'Cadou');
CALL ISTORIC_TRANZACTII(20002, CURRENT_DATE(), CURRENT_DATE(),1);


CALL ISTORIC_TRANZACTII(10001, CURRENT_DATE(), CURRENT_DATE(),1);
CALL EXTRAGERE_ATM(10001, 500);
CALL EXTRAGERE_ATM(10001, 500);
CALL EXTRAGERE_ATM(10001, 30000);
CALL EXTRAGERE_ATM(10009, 300);
CALL ISTORIC_TRANZACTII(10009, CURRENT_DATE(), CURRENT_DATE(),0);

CALL ALIMENTARE_CONT_ATM(20002,2000);

CALL BALANTA_CONT(20002, CURRENT_DATE(), NOW());
CALL ISTORIC_TRANZACTII(20002, CURRENT_DATE(), CURRENT_DATE(),0);
CALL BALANTA_CONT(20002, CURRENT_DATE(), '12:09');

CALL EXTRAGERE_GHISEU(20002, 500);

CALL INCHIDERE_CONT(20002);
CALL ISTORIC_TRANZACTII(20002, CURRENT_DATE(), CURRENT_DATE(),0);
select * from cont;
CALL BALANTA_CONT(20002, CURRENT_DATE(), NOW());